---
description: "PostgreSQL constraints beyond basics: named CHECKs, UNIQUE NULLS NOT DISTINCT, exclusion constraints, deferrable keys, NOT VALID and generated columns."
---

# 06 · Constraints Beyond the Basics

Application code validates data on the way in. Constraints validate it **always**: from every
service, every migration script, every analyst with write access, every bug. A rule enforced only
in code is a rule that will eventually be broken by something that is not that code.

You know `PRIMARY KEY`, `NOT NULL`, `UNIQUE` and `FOREIGN KEY`. PostgreSQL goes much further —
exclusion constraints in particular can express rules that most databases leave to fragile
application locking. This lesson covers the features, and the operational tricks for adding
constraints to tables that already hold data.

All examples run in a database called `lab` on PostgreSQL 18.6.

## Name your CHECK constraints

```sql
CREATE TABLE products (
  id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  sku        text NOT NULL UNIQUE,
  price      numeric(10,2) NOT NULL CHECK (price >= 0),
  sale_price numeric(10,2),
  CONSTRAINT sale_below_price CHECK (sale_price IS NULL OR sale_price < price)
);

INSERT INTO products (sku, price, sale_price) VALUES ('A1', 10, 12);
```

```text
ERROR:  new row for relation "products" violates check constraint "sale_below_price"
DETAIL:  Failing row contains (1, A1, 10.00, 12.00).
```

A named constraint makes the error self-explanatory, and your application can map the name to a
user-facing message. Unnamed constraints get generated names like `products_price_check`.

Two rules for CHECK expressions:

1. **A CHECK passes when the expression is true *or NULL*.** `CHECK (sale_price < price)` would let
   a NULL `sale_price` through — which here is what we want, but be deliberate about it.
2. **They must be immutable and look only at the current row.** No subqueries, no `now()`
   comparisons that change meaning over time, no looking at other rows. Cross-row rules need
   UNIQUE, EXCLUDE, foreign keys or triggers.

## UNIQUE and NULL

By the SQL standard, NULL is not equal to NULL, so a unique constraint allows any number of rows
with NULL in the key:

```sql
CREATE TABLE coupons (code text, region text, UNIQUE (code, region));
INSERT INTO coupons VALUES ('SAVE10', NULL), ('SAVE10', NULL);   -- INSERT 0 2
```

If "no region" should count as one value, PostgreSQL 15 added `NULLS NOT DISTINCT`:

```sql
CREATE TABLE coupons2 (code text, region text, UNIQUE NULLS NOT DISTINCT (code, region));
INSERT INTO coupons2 VALUES ('SAVE10', NULL), ('SAVE10', NULL);
```

```text
ERROR:  duplicate key value violates unique constraint "coupons2_code_region_key"
DETAIL:  Key (code, region)=(SAVE10, null) already exists.
```

On older versions the usual workaround is a unique index on `(code, coalesce(region, ''))` or a
partial unique index.

### Partial uniqueness

"Only one *active* subscription per user" is a unique index with a `WHERE` clause:

```sql
CREATE UNIQUE INDEX one_active_sub ON subscriptions (user_id) WHERE status = 'active';
```

Cancelled subscriptions do not participate. This cannot be written as a table constraint, only as
an index — but it enforces uniqueness exactly the same way.

## Exclusion constraints: no overlapping bookings

A unique constraint says "no two rows have *equal* keys". An exclusion constraint generalises it:
"no two rows where *these operators all return true*". With ranges and the overlap operator `&&`,
that is "no two bookings of the same room overlap":

```sql
CREATE EXTENSION btree_gist;   -- lets a GiST index handle plain = on integers

CREATE TABLE bookings (
  id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  room_id int NOT NULL,
  during  tstzrange NOT NULL,
  EXCLUDE USING gist (room_id WITH =, during WITH &&)
);

INSERT INTO bookings (room_id, during) VALUES (101, '[2026-11-02 09:00, 2026-11-02 10:00)');
INSERT INTO bookings (room_id, during) VALUES (101, '[2026-11-02 10:00, 2026-11-02 11:00)');
INSERT INTO bookings (room_id, during) VALUES (101, '[2026-11-02 10:30, 2026-11-02 11:30)');
```

```text
INSERT 0 1
INSERT 0 1
ERROR:  conflicting key value violates exclusion constraint "bookings_room_id_during_excl"
DETAIL:  Key (room_id, during)=(101, ["2026-11-02 10:30:00+05:30","2026-11-02 11:30:00+05:30"))
conflicts with existing key (room_id, during)=(101, ["2026-11-02 10:00:00+05:30","2026-11-02 11:00:00+05:30")).
```

The 09:00–10:00 and 10:00–11:00 bookings coexist because `[ )` ranges exclude the upper bound. The
10:30 booking overlaps and is rejected. The same booking for room 102 succeeds.

Why this matters: doing the same check in application code ("SELECT overlapping bookings; if none,
INSERT") has a race. Two requests both see no conflict and both insert. The exclusion constraint is
enforced by the index under the hood, so concurrent inserts are serialised correctly — the second
one waits for the first to commit or roll back, then fails.

PostgreSQL 18 also lets primary keys and unique constraints use `WITHOUT OVERLAPS` on a range column
— a temporal key that builds the same kind of GiST-backed check with standard syntax.

## Deferrable constraints

Normal (non-deferrable) unique constraints are checked **row by row** while a statement runs. Swap two
positions in one statement:

```sql
CREATE TABLE playlist (pos int UNIQUE, song text);
INSERT INTO playlist VALUES (1,'a'), (2,'b');
UPDATE playlist SET pos = CASE pos WHEN 1 THEN 2 ELSE 1 END;
```

```text
ERROR:  duplicate key value violates unique constraint "playlist_pos_key"
DETAIL:  Key (pos)=(2) already exists.
```

The first row updated to 2 while the other row still had 2. Declaring the constraint `DEFERRABLE`
changes when it is checked:

```sql
CREATE TABLE playlist2 (pos int, song text,
  CONSTRAINT pos_uq UNIQUE (pos) DEFERRABLE INITIALLY IMMEDIATE);
INSERT INTO playlist2 VALUES (1,'a'), (2,'b');
UPDATE playlist2 SET pos = CASE pos WHEN 1 THEN 2 ELSE 1 END;   -- UPDATE 2
```

`INITIALLY IMMEDIATE` checks at the **end of each statement**, so the one-statement swap works. To
spread changes over several statements, defer to commit:

```sql
BEGIN;
SET CONSTRAINTS pos_uq DEFERRED;
UPDATE playlist2 SET pos = 1 WHERE song = 'a';   -- temporarily two rows with pos 1
UPDATE playlist2 SET pos = 2 WHERE song = 'b';
COMMIT;                                          -- checked here: OK
```

If the data is still wrong at commit, the whole transaction fails:

```text
UPDATE playlist2 SET pos = 3 WHERE song = 'a';
UPDATE playlist2 SET pos = 3 WHERE song = 'b';
COMMIT;
ERROR:  duplicate key value violates unique constraint "pos_uq"
DETAIL:  Key (pos)=(3) already exists.
```

Deferred foreign keys solve the chicken-and-egg problem of inserting rows that reference each other.
The cost: deferrable unique constraints cannot be the target of a foreign key, and the planner
cannot use them to prove uniqueness, so only make constraints deferrable when you need it.

## Foreign keys: actions and the index you must add

```sql
CREATE TABLE orders (
  id          bigint PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES customers (id) ON DELETE RESTRICT
);
CREATE INDEX ON orders (customer_id);   -- PostgreSQL does NOT create this for you
```

`ON DELETE` choices: `NO ACTION` (default; error at end of statement, can be deferred), `RESTRICT`
(error immediately), `CASCADE` (delete children), `SET NULL`, `SET DEFAULT`. Use `CASCADE` for true
ownership (order → line items), never as a convenience on things like customers.

PostgreSQL indexes the *referenced* side (it must be a primary key or unique) but **not the
referencing column**. Without an index on `orders.customer_id`, every delete from `customers` scans
all of `orders` to check for references. That is the single most common missing index in
PostgreSQL schemas.

## Adding constraints to tables that already have data

Adding a foreign key normally scans the whole table to check existing rows — and fails if any are bad:

```sql
INSERT INTO orders VALUES (10, 1), (11, 999);    -- 999 is not a customer
ALTER TABLE orders ADD CONSTRAINT orders_customer_fk
  FOREIGN KEY (customer_id) REFERENCES customers (id);
```

```text
ERROR:  insert or update on table "orders" violates foreign key constraint "orders_customer_fk"
DETAIL:  Key (customer_id)=(999) is not present in table "customers".
```

`NOT VALID` adds the constraint **without checking existing rows**, but enforces it for every new
or updated row from that moment:

```sql
ALTER TABLE orders ADD CONSTRAINT orders_customer_fk
  FOREIGN KEY (customer_id) REFERENCES customers (id) NOT VALID;      -- instant
INSERT INTO orders VALUES (12, 998);
```

```text
ERROR:  insert or update on table "orders" violates foreign key constraint "orders_customer_fk"
DETAIL:  Key (customer_id)=(998) is not present in table "customers".
```

Then clean up and validate as a separate step:

```text
ALTER TABLE orders VALIDATE CONSTRAINT orders_customer_fk;
ERROR:  ... Key (customer_id)=(999) is not present in table "customers".

DELETE FROM orders WHERE id = 11;
ALTER TABLE orders VALIDATE CONSTRAINT orders_customer_fk;
ALTER TABLE
```

On a big table this split matters enormously: `ADD ... NOT VALID` takes a brief lock, and
`VALIDATE` scans the table under a weaker lock that does not block reads or writes. Level 4 · 07
builds whole migrations around it. `NOT VALID` works for CHECK and foreign keys, and since
PostgreSQL 18 for `NOT NULL` too:

```sql
ALTER TABLE customers ADD CONSTRAINT name_nn NOT NULL name NOT VALID;
SELECT conname, contype, convalidated FROM pg_constraint WHERE conrelid = 'customers'::regclass;
```

```text
        conname        | contype | convalidated
-----------------------+---------+--------------
 customers_id_not_null | n       | t
 customers_pkey        | p       | t
 name_nn               | n       | f
```

(PostgreSQL 18 also records NOT NULL constraints in `pg_constraint` with names, as shown.)

## Generated columns

```sql
CREATE TABLE line_items (
  qty        int NOT NULL CHECK (qty > 0),
  unit_price numeric(10,2) NOT NULL,
  total      numeric(12,2) GENERATED ALWAYS AS (qty * unit_price) STORED,
  total_v    numeric(12,2) GENERATED ALWAYS AS (qty * unit_price)
);
INSERT INTO line_items (qty, unit_price) VALUES (3, 2.50);
SELECT * FROM line_items;
```

```text
 qty | unit_price | total | total_v
-----+------------+-------+---------
   3 |       2.50 |  7.50 |    7.50
```

A **stored** generated column is computed on write and occupies disk space; it can be indexed. A
**virtual** generated column — the default when you omit `STORED`, new in PostgreSQL 18 — is
computed on read and takes no space. Neither can be written directly:

```text
ERROR:  cannot insert a non-DEFAULT value into column "total"
DETAIL:  Column "total" is a generated column.
```

Use generated columns for derived values that must never drift from their inputs: totals,
normalised search keys (`lower(email)`), extracted JSON fields you want to index.

## How It Actually Works

Constraints live in `pg_constraint` (`contype`: `p` primary, `u` unique, `f` foreign, `c` check,
`x` exclusion, `n` not-null). Their enforcement uses three different mechanisms:

- **CHECK and NOT NULL** are evaluated by the executor on every inserted or updated tuple, right
  before it is written. Cheap and local.
- **UNIQUE, PRIMARY KEY and EXCLUDE** are enforced by an **index**. When a new entry is inserted
  the index access method looks for conflicting entries; if it finds one belonging to a transaction
  still in progress, the inserting backend *waits* for that transaction to finish, then re-checks.
  This is what makes them race-free. Deferrable versions insert the entry anyway, remember the
  potential conflict, and re-check it at statement end or commit.
- **FOREIGN KEY** is implemented with internal system triggers on both tables. Inserting a child
  row runs a `SELECT 1 FROM parent WHERE id = $1 FOR KEY SHARE` — locking the parent row against
  deletion — and deleting a parent row runs a query against the child table. That second query is
  why the missing index on the referencing column hurts.

`NOT VALID` simply sets `convalidated = false`, which tells the system to skip the initial scan;
the triggers or checks are active immediately. `VALIDATE CONSTRAINT` performs the scan later and
flips the flag.

## Common mistakes

- Forgetting the index on foreign-key columns.
- Relying on application "check then insert" logic for uniqueness or overlaps instead of a
  constraint.
- Forgetting that CHECK constraints pass on NULL.
- `ON DELETE CASCADE` on relationships that are not ownership — one delete removes far more than
  intended.
- Adding a validated constraint to a large, busy table in one step during peak traffic.
- Making constraints deferrable "just in case", losing planner optimisations and FK targets.

## Exercise

1. Create a `meeting_rooms` / `reservations` schema where reservations for the same room cannot
   overlap, a reservation must be at least 15 minutes and at most 8 hours long, and a cancelled
   reservation (status `cancelled`) does not block new ones. (Hint: an exclusion constraint can
   have a `WHERE` clause.)
2. Prove the overlap rule with two concurrent `psql` sessions: insert overlapping rows inside two
   open transactions and observe the second one wait, then fail when the first commits.
3. Load 10,000 rows containing a few invalid foreign-key references into a child table, add the
   foreign key as `NOT VALID`, find and fix the bad rows, then validate it.
4. Add a stored generated column `email_normalized` = `lower(trim(email))` to a users table, put a
   unique index on it, and show that `Ana@X.com` and ` ana@x.com` now conflict.
