---
description: "PostgreSQL SQL features that remove application code: RETURNING (old/new), ON CONFLICT upserts, MERGE, DISTINCT ON, FILTER, generate_series and COPY."
---

# 08 · PostgreSQL-Flavoured SQL: RETURNING, Upserts, MERGE & More

Standard SQL gets you a long way, but PostgreSQL adds features that replace whole chunks of
application code — and, more importantly, replace *multi-statement* application logic that has
race conditions with single statements that do not. This lesson is a tour of the ones you will use
every week.

Examples run in database `lab` on PostgreSQL 18.6.

```sql
CREATE TABLE inventory (
  sku        text PRIMARY KEY,
  qty        int NOT NULL,
  updated_at timestamptz NOT NULL DEFAULT now()
);
```

## RETURNING: get back what you changed

```sql
INSERT INTO inventory (sku, qty) VALUES ('A1', 5), ('B2', 0) RETURNING sku, qty;
```

```text
 sku | qty
-----+-----
 A1  |   5
 B2  |   0
(2 rows)

INSERT 0 2
```

`RETURNING` works on `INSERT`, `UPDATE`, `DELETE` and `MERGE`. It saves a round trip, and — more
importantly — it returns *exactly* the rows that statement touched, including generated IDs and
defaults, with no window for another session to change them in between.

PostgreSQL 18 lets you return both the before and after values of an update:

```sql
UPDATE inventory SET qty = qty - 2 WHERE sku = 'A1'
RETURNING old.qty AS before, new.qty AS after;
```

```text
 before | after
--------+-------
      8 |     6
```

Before version 18 you needed a CTE or a trigger to capture the old value.

## Upserts with ON CONFLICT

"Insert, or update if it already exists" is a race in application code: two requests both check,
both find nothing, both insert, one fails. `ON CONFLICT` makes it a single atomic operation:

```sql
INSERT INTO inventory (sku, qty) VALUES ('A1', 3), ('C3', 7)
ON CONFLICT (sku) DO UPDATE
  SET qty = inventory.qty + EXCLUDED.qty,
      updated_at = now()
RETURNING sku, qty, (xmax <> 0) AS was_update;
```

```text
 sku | qty | was_update
-----+-----+------------
 A1  |   8 | t
 C3  |   7 | f
```

- `ON CONFLICT (sku)` names the **conflict target**: a unique index or constraint. You can also name
  a constraint directly (`ON CONFLICT ON CONSTRAINT inventory_pkey`) or target a partial unique
  index by repeating its `WHERE`.
- `EXCLUDED` is the row you *tried* to insert; the table name refers to the existing row.
- Add `WHERE` to the `DO UPDATE` to skip pointless updates:
  `... DO UPDATE SET qty = EXCLUDED.qty WHERE inventory.qty IS DISTINCT FROM EXCLUDED.qty`.
- `(xmax <> 0)` is a well-known trick for telling inserts from updates; it relies on an
  implementation detail (lesson Level 2 · 01 explains `xmax`), so treat it as a debugging aid.

`DO NOTHING` silently skips conflicts — and returns nothing for them:

```text
INSERT INTO inventory (sku, qty) VALUES ('A1', 100) ON CONFLICT DO NOTHING RETURNING sku;
 sku
-----
(0 rows)
```

Code that does `INSERT ... ON CONFLICT DO NOTHING RETURNING id` and expects an ID back will get an
empty result when the row already existed; you then need a separate `SELECT`.

One upsert statement cannot touch the same row twice:

```text
INSERT INTO inventory (sku, qty) VALUES ('A1', 1), ('A1', 2)
ON CONFLICT (sku) DO UPDATE SET qty = EXCLUDED.qty;
ERROR:  ON CONFLICT DO UPDATE command cannot affect row a second time
HINT:  Ensure that no rows proposed for insertion within the same command have duplicate constrained values.
```

Deduplicate the batch first (for example with `DISTINCT ON`, below).

## MERGE: synchronise a table from a source

`MERGE` (PostgreSQL 15+) applies different actions depending on whether each source row matches.
Here a supplier feed sets stock levels, and a zero quantity means "discontinued":

```sql
CREATE TABLE stock_feed (sku text, qty int);
INSERT INTO stock_feed VALUES ('A1', 10), ('B2', 0), ('D4', 4);

MERGE INTO inventory i
USING stock_feed f ON i.sku = f.sku
WHEN MATCHED AND f.qty = 0 THEN DELETE
WHEN MATCHED THEN UPDATE SET qty = f.qty, updated_at = now()
WHEN NOT MATCHED THEN INSERT (sku, qty) VALUES (f.sku, f.qty)
RETURNING merge_action(), f.sku, i.qty;
```

```text
 merge_action | sku | qty
--------------+-----+-----
 UPDATE       | A1  |  10
 DELETE       | B2  |   0
 INSERT       | D4  |   4
(3 rows)

MERGE 3
```

`RETURNING` and `merge_action()` arrived in PostgreSQL 17. PostgreSQL 17 also added
`WHEN NOT MATCHED BY SOURCE`, for rows in the target that the source does not mention — handy for
"delete everything the feed no longer lists".

**MERGE or ON CONFLICT?** They solve different problems. `ON CONFLICT` is designed for concurrency:
it uses the unique index to resolve a race with another inserting session. `MERGE` decides
"matched or not" by joining at the start; if a concurrent session inserts a matching row after that,
your `MERGE` can still try to insert and hit a unique violation. Use `ON CONFLICT` for
high-concurrency upserts from an application, and `MERGE` for batch synchronisation where you
control the concurrency (ETL jobs, nightly loads).

## UPDATE ... FROM and DELETE ... USING

Join other tables into an update or delete without subqueries:

```sql
UPDATE inventory i SET qty = i.qty + f.qty
FROM stock_feed f
WHERE f.sku = i.sku
RETURNING i.sku, i.qty;

DELETE FROM inventory i USING stock_feed f
WHERE f.sku = i.sku AND f.qty = 4
RETURNING i.sku;
```

Careful: if the `FROM` side contains *several* rows matching one target row, `UPDATE ... FROM`
updates the row once using an arbitrary one of them. Make sure the join is one-to-one.

## DISTINCT ON: the first row per group

"The latest reading per sensor" is a top-1-per-group query. In PostgreSQL:

```sql
SELECT DISTINCT ON (sensor) sensor, taken_at, value
FROM readings
ORDER BY sensor, taken_at DESC;
```

```text
 sensor |         taken_at          | value
--------+---------------------------+-------
 s1     | 2026-10-01 11:00:00+05:30 |  21.4
 s2     | 2026-10-01 12:00:00+05:30 |  17.2
```

`DISTINCT ON (sensor)` keeps the first row of each `sensor` group in `ORDER BY` order. The
`ORDER BY` must start with the `DISTINCT ON` expressions, and the rest of it decides which row
"wins". With an index on `(sensor, taken_at DESC)` this is fast. The portable alternative is a
`row_number()` window function filtered to 1.

## FILTER: conditional aggregates

```sql
SELECT count(*)                                  AS total,
       count(*) FILTER (WHERE value > 19)         AS warm,
       avg(value) FILTER (WHERE sensor = 's1')    AS s1_avg
FROM readings;
```

```text
 total | warm |       s1_avg
-------+------+---------------------
     5 |    3 | 20.4666666666666667
```

Cleaner than `sum(CASE WHEN ... THEN 1 ELSE 0 END)`, and it works with every aggregate, including
`array_agg` and `string_agg`. Both of those accept their own `ORDER BY`:
`string_agg(sku, ', ' ORDER BY sku)`.

## generate_series: rows from nothing

Reports usually need a row for every day, even days with no data:

```sql
SELECT d::date AS day, count(r.*) AS readings
FROM generate_series('2026-09-29'::date, '2026-10-02'::date, interval '1 day') d
LEFT JOIN readings r ON r.taken_at::date = d::date
GROUP BY d ORDER BY d;
```

```text
    day     | readings
------------+----------
 2026-09-29 |        0
 2026-09-30 |        0
 2026-10-01 |        5
 2026-10-02 |        0
```

`count(r.*)` counts only matched rows, so empty days show 0 rather than 1. `generate_series` is also
the quickest way to create test data — this course uses it constantly.

## COPY: bulk loading at full speed

How you load data matters more than almost any tuning setting. Loading the same 100,000 two-column
rows from Python (psycopg 3.3) to a local server on the Mac used to write this course, one run each:

```text
single-row INSERTs, autocommit each         8.02 s
single-row INSERTs, one transaction         4.72 s
executemany (pipelined), one transaction    1.09 s
COPY FROM STDIN                             0.03 s
```

Your numbers will differ with hardware, network latency and settings, but the ordering is robust:

- **Autocommit per row** pays for a commit — a WAL flush — on every row.
- **One transaction** removes the flushes but still pays a network round trip and a parse/plan per
  statement.
- **Pipelining** (`executemany` in psycopg 3 sends statements without waiting for each reply) removes
  most of the round-trip cost.
- **COPY** streams rows in a compact format through a single command: no per-row statement at all.

```python
with conn.cursor() as cur, cur.copy("COPY events (id, payload) FROM STDIN") as cp:
    for row in rows:
        cp.write_row(row)
```

Every serious driver exposes COPY (psycopg's `copy()`, JDBC's `CopyManager`, node-postgres's
`pg-copy-streams`, Go's `pgx.CopyFrom`). For huge loads into a new table, also create indexes and
foreign keys *after* loading.

## How It Actually Works

**ON CONFLICT** uses "speculative insertion". The executor first checks the conflict target's unique
index for an existing key. If none, it inserts the heap tuple marked as speculative, then inserts
the index entry; if another session raced in with the same key in between, the index insertion
detects it, the speculative tuple is killed, and the operation loops back to the "row exists" path.
When a row exists, `DO UPDATE` locks it (like `SELECT FOR UPDATE`) and applies the update to the
latest committed version — even under Read Committed it never sees "nothing" and "something" at
once. That loop is why upserts are race-free without any explicit locking.

**MERGE** is planned as a join between source and target (an outer join when there are `NOT MATCHED`
clauses). Each joined row is classified matched/not matched, the first `WHEN` clause whose condition
holds is applied, and concurrent updates to matched rows are handled like an `UPDATE` (waiting and
re-checking). It has no speculative-insert path, which is the root of the concurrency difference.

**RETURNING** is evaluated by the same executor node that modifies the row, on the new tuple (or,
for `old.`, on the tuple version being replaced), so it sees exactly what was written, including
defaults, generated columns and values set by `BEFORE` triggers.

**COPY** bypasses the parser and planner per row: the server reads a stream, converts each field with
the column type's input function, and inserts tuples in batches using a multi-insert path that
writes several tuples per heap page operation and WAL record.

## Common mistakes

- Check-then-insert in application code instead of `ON CONFLICT`.
- Expecting `RETURNING` to return rows skipped by `ON CONFLICT DO NOTHING`.
- Using `MERGE` for concurrent upserts and being surprised by unique violations.
- `UPDATE ... FROM` with a one-to-many join, updating each row with an arbitrary match.
- `DISTINCT ON` with an `ORDER BY` that does not start with the same expressions (an error), or
  without a tie-breaker (non-deterministic winner).
- Loading data row by row with autocommit.

## Exercise

1. Create a `page_views (page text PRIMARY KEY, views bigint)` table and write a single statement that
   increments a page's counter, creating it on first view. Run it concurrently from two `psql`
   sessions inside open transactions and observe what the second one does.
2. Write a `MERGE` that synchronises a `products` table from a `product_feed` staging table:
   update changed prices, insert new products, and mark products missing from the feed as inactive
   using `WHEN NOT MATCHED BY SOURCE`.
3. Using `DISTINCT ON`, return each customer's most recent order; then write the same query with
   `row_number()` and compare the plans with `EXPLAIN`.
4. Generate a calendar of the next 30 days with `generate_series` and left-join your bookings to show
   free days.
5. Load a 1-million-row CSV with `\copy` and with single `INSERT`s in one transaction; record your own
   timings.
