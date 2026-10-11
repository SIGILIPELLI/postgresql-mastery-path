---
description: "Choose the right PostgreSQL types: numeric vs float, text vs varchar, timestamptz, intervals, identity columns, uuidv7, enums, arrays and range types."
---

# 03 · PostgreSQL Data Types That Matter

PostgreSQL has more built-in types than any other mainstream database — geometric shapes, network
addresses, ranges, multiranges, full-text vectors. You do not need all of them. You *do* need to
make five or six decisions correctly on every table you design, because a wrong column type is one
of the most expensive mistakes to fix once a table holds a billion rows.

This lesson covers those decisions. Every result below came from PostgreSQL 18.6.

## Numbers: exact or approximate?

```sql
SELECT 0.1::float8 + 0.2::float8 AS float_sum,
       0.1::numeric + 0.2::numeric AS numeric_sum;
```

```text
      float_sum      | numeric_sum
---------------------+-------------
 0.30000000000000004 |         0.3
(1 row)
```

`real` and `double precision` (`float4`/`float8`) are IEEE 754 binary floating point: fast, compact
and *approximate*. `(0.1 + 0.2) = 0.3` is false for them. `numeric` is exact decimal arithmetic in
software — slower, but correct to the last digit.

| Use case | Type |
|----------|------|
| Money, quantities with fixed decimals, anything audited | `numeric(12,2)` or integer cents in `bigint` |
| Measurements, scientific data, ML features | `double precision` |
| Counters, IDs | `integer` or `bigint` |

Avoid the `money` type: its formatting depends on the `lc_monetary` setting, so the same value can
print differently on two servers, and it has no currency.

Integers fail loudly rather than wrapping:

```text
SELECT 2147483647::int + 1;
ERROR:  integer out of range
```

That number — 2,147,483,647 — is a real ceiling. Tables with a `serial`/`integer` primary key do
reach it, and the fix (changing the column to `bigint`) rewrites the whole table under an exclusive
lock. **Use `bigint` for primary keys** on any table that might grow; the extra 4 bytes per row is
cheap insurance.

```sql
SELECT pg_column_size(1::smallint) s, pg_column_size(1::int) i,
       pg_column_size(1::bigint) b,   pg_column_size(1::numeric) n,
       pg_column_size(123456789.123456::numeric) n2;
```

```text
 s | i | b | n | n2
---+---+---+---+----
 2 | 4 | 8 | 8 | 16
(1 row)
```

`numeric` is variable-length: its size grows with the number of digits.

## Text: just use `text`

In PostgreSQL, `text`, `varchar` and `varchar(n)` are stored identically. There is no performance
benefit to `varchar(255)` — it is a habit carried over from other databases. A length limit is a
*constraint*, and like any constraint it should express a real business rule.

Watch how `varchar(n)` behaves:

```text
SELECT 'abc'::varchar(2);
 varchar
---------
 ab

INSERT INTO t (code) VALUES ('abcd');      -- column is varchar(3)
ERROR:  value too long for type character varying(3)
```

An insert that is too long fails, but an explicit cast **silently truncates**. If you need a limit,
`text` with `CHECK (char_length(code) <= 3)` behaves consistently and can be changed later without
touching the type.

Avoid `char(n)`: it pads with spaces, and the padding causes surprising comparisons.

Length is measured in characters, not bytes:

```text
SELECT 'naïve' AS s, length('naïve') AS chars, octet_length('naïve') AS bytes;
   s   | chars | bytes
-------+-------+-------
 naïve |     5 |     6
```

## Time: `timestamptz` almost always

This is the decision most often made wrong. Two columns, same input:

```sql
SET timezone = 'UTC';
CREATE TEMP TABLE times (ts timestamp, tstz timestamptz);
INSERT INTO times VALUES ('2026-03-29 09:00', '2026-03-29 09:00');
SELECT * FROM times;
```

```text
         ts          |          tstz
---------------------+------------------------
 2026-03-29 09:00:00 | 2026-03-29 09:00:00+00
```

```sql
SET timezone = 'Asia/Kolkata';
SELECT * FROM times;
```

```text
         ts          |           tstz
---------------------+---------------------------
 2026-03-29 09:00:00 | 2026-03-29 14:30:00+05:30
```

`timestamptz` ("timestamp with time zone") does **not** store a time zone. It stores an absolute
instant (microseconds since 2000-01-01 UTC) and converts to the session's `timezone` for display.
`timestamp` ("without time zone") stores a wall-clock reading with no idea where it was taken. When
the session time zone changed, `ts` stayed "09:00" — but 09:00 *where*? Nobody knows any more.

Use `timestamptz` for anything that happened at a moment: created_at, paid_at, log events. Use
`timestamp` only for genuinely zone-less concepts like "the store opens at 09:00 local time
wherever the store is", and store the zone in another column. Use `date` when there is no time of
day.

### Interval arithmetic is calendar-aware

```sql
SET timezone = 'Europe/London';
SELECT '2026-03-28 12:00'::timestamptz + interval '1 day'    AS plus_1_day,
       '2026-03-28 12:00'::timestamptz + interval '24 hours' AS plus_24_hours;
```

```text
       plus_1_day       |     plus_24_hours
------------------------+------------------------
 2026-03-29 12:00:00+01 | 2026-03-29 13:00:00+01
```

The UK moved its clocks forward on 29 March 2026. "One day later" means same wall-clock time
tomorrow; "24 hours later" means exactly 86,400 seconds later. Both are correct; they answer
different questions. Month arithmetic clamps to the end of the month:

```text
SELECT date '2026-01-31' + interval '1 month';
 2026-02-28 00:00:00
```

## Primary keys: identity columns and UUIDs

`serial` is the old way: it creates a sequence and a default, loosely attached to the column. Since
PostgreSQL 10, the SQL-standard **identity column** is preferred:

```sql
CREATE TABLE items (
  id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name text
);
INSERT INTO items (id, name) VALUES (42, 'manual');
```

```text
ERROR:  cannot insert a non-DEFAULT value into column "id"
DETAIL:  Column "id" is an identity column defined as GENERATED ALWAYS.
HINT:  Use OVERRIDING SYSTEM VALUE to override.
```

`GENERATED ALWAYS` protects you from application code inventing IDs that will later collide with
the sequence. (`GENERATED BY DEFAULT` allows explicit values, which is occasionally needed for data
migrations.) The identity's sequence is owned by the table, so dropping or copying the table with
`CREATE TABLE ... (LIKE ... INCLUDING IDENTITY)` behaves sensibly.

Sequences never go backwards, even on rollback:

```sql
INSERT INTO items (id, name) OVERRIDING SYSTEM VALUE VALUES (42, 'manual');
BEGIN; INSERT INTO items (name) VALUES ('rolled back'); ROLLBACK;
INSERT INTO items (name) VALUES ('kept') RETURNING id;
```

```text
 id
----
  2
```

Value 1 was consumed by the rolled-back insert. **Gaps in IDs are normal** and you should never
rely on IDs being contiguous. (And note the trap just set: the sequence will reach 42 eventually
and collide with the manually inserted row. After bulk-loading explicit IDs, move the sequence with
`ALTER TABLE items ALTER COLUMN id RESTART WITH 1000`.)

### UUIDs

When IDs must be generated outside the database or must not reveal row counts, use `uuid` (16 bytes,
stored in binary — never store UUIDs as `text`). PostgreSQL 18 added `uuidv7()`:

```text
SELECT uuidv7() AS v7, gen_random_uuid() AS v4;
                  v7                  |                  v4
--------------------------------------+--------------------------------------
 01a129a5-50b1-7c7b-bb43-8180d4f79fca | 42e06896-8eb1-4627-97fc-2b5018f07700
```

A version 4 UUID is fully random, so consecutive inserts land on random pages of the primary key
index. A version 7 UUID starts with a millisecond timestamp, so new keys are roughly increasing and
inserts append to the right edge of the index like a sequence does — much friendlier to the buffer
cache on large tables. The trade-off: a v7 UUID reveals when the row was created
(`uuid_extract_timestamp()` reads it back). On older versions, `gen_random_uuid()` (v4) is
built in since PostgreSQL 13; v7 needs an extension or application-side generation.

## Enums: fixed, ordered sets

```sql
CREATE TYPE order_status AS ENUM ('pending', 'paid', 'shipped');
SELECT 'paid'::order_status < 'shipped'::order_status;   -- t: enums sort in declared order
SELECT 'refunded'::order_status;
```

```text
ERROR:  invalid input value for enum order_status: "refunded"
```

```sql
ALTER TYPE order_status ADD VALUE 'refunded' AFTER 'shipped';
SELECT enum_range(NULL::order_status);
```

```text
           enum_range
---------------------------------
 {pending,paid,shipped,refunded}
```

Enums take 4 bytes and validate input. But you can add values and rename them, not remove them. If
the list changes often or carries extra data (a label, a sort order, an "is active" flag), use a
small lookup table with a foreign key instead.

## Arrays

```text
SELECT ARRAY['sql','pg'] AS tags, (ARRAY['sql','pg'])[1] AS first_is_1, (ARRAY['sql','pg'])[0] AS index_zero;
   tags   | first_is_1 | index_zero
----------+------------+------------
 {sql,pg} | sql        |
```

Arrays are **1-based**, and an out-of-range subscript returns NULL instead of an error. Test
membership with `= ANY(...)` or containment with `@>`; the latter can use a GIN index (Level 2 · 05).

Arrays suit small, self-contained lists such as tags. If you ever need to join on an element,
enforce a foreign key on it, or update one element under concurrency, you wanted a child table.

## Ranges and multiranges

A range is a single value holding a lower and upper bound:

```text
SELECT int4range(10, 20) AS r, int4range(10,20) @> 15 AS has15,
       int4range(10,20) @> 20 AS has20,
       tstzrange('2026-01-01','2026-01-02') && tstzrange('2026-01-01 12:00','2026-01-03') AS overlaps;
    r    | has15 | has20 | overlaps
---------+-------+-------+----------
 [10,20) | t     | f     | t
```

`[10,20)` means inclusive lower, exclusive upper — the convention that makes adjacent ranges such
as booking slots fit together without overlap. Multiranges (PostgreSQL 14+) hold several
non-overlapping ranges and merge automatically:

```text
SELECT int4multirange(int4range(1,5), int4range(3,8), int4range(10,12));
 {[1,8),[10,12)}
```

Ranges combine with exclusion constraints to make "no two bookings of the same room overlap" a rule
the database enforces — lesson 6 builds exactly that.

## The rest, briefly

- `boolean` accepts `'t'`, `'yes'`, `'on'`, `'1'` and their opposites as input.
- `bytea` for binary data (`'\xDEADBEEF'::bytea` has length 4). Large files belong in object
  storage with the path in the database.
- `inet` and `cidr` for IP addresses, with operators such as `<<=` ("is contained in subnet").
- `jsonb` for semi-structured data — a whole lesson in Level 3.

## How It Actually Works

Every type in PostgreSQL is a row in the `pg_type` catalog that points to C functions for input
(text → internal), output (internal → text), and optionally binary send/receive. That is why
`'2026-03-29'::date` works: the literal is an untyped string until the input function for `date`
parses it. It is also why extensions can add types that behave exactly like built-ins.

Types are either **fixed-length** (`int4` is always 4 bytes, `timestamptz` always 8) or
**variable-length** ("varlena": `text`, `numeric`, `bytea`, arrays, `jsonb`). A varlena value carries
a 1- or 4-byte length header, and once a row gets large, big varlena values are compressed or moved
out of line into a TOAST table (Level 2 · 03). Fixed-length values also have **alignment**
requirements: an 8-byte `bigint` must start at an 8-byte boundary, so a `boolean` followed by a
`bigint` wastes 7 bytes of padding. On very wide, very tall tables, ordering columns from largest
fixed-width to smallest can save measurable space.

`timestamptz` is an 8-byte integer count of microseconds from 2000-01-01 00:00 UTC. Conversion to
and from local time uses the IANA time-zone database compiled into PostgreSQL (or the operating
system's copy), which is why it correctly knows the UK's 2026 clock change.

## Common mistakes

- Using `float` for money.
- `integer` primary keys on tables that will grow; migrating to `bigint` later is a full rewrite.
- `timestamp` without time zone for event times, then discovering mixed zones in the data.
- `varchar(255)` everywhere "for performance" — there is none — and silent truncation on casts.
- Storing UUIDs, dates or numbers as `text`: you lose validation, compact storage and correct
  sorting (`'10' < '9'` as text).
- Expecting IDs without gaps.

## Exercise

1. Create an `invoices` table with: a `bigint` identity key, a `uuid` public ID defaulting to
   `uuidv7()` (or `gen_random_uuid()` before PostgreSQL 18), an amount that cannot lose cents, a
   currency code limited to exactly 3 uppercase letters with a `CHECK`, and an issue time.
2. Insert three rows, then change your session's `timezone` twice and observe how the issue time
   displays.
3. Compute the due date as "issued + 1 month" for an invoice issued on 31 January. Is the result what
   your billing team would expect? Write down the rule you would use instead.
4. Create an enum for invoice status. Try to remove a value. What options do you have?
5. Use `pg_column_size` on a row (`SELECT pg_column_size(i.*) FROM invoices i`) and experiment with
   column order to see whether padding changes the size.
