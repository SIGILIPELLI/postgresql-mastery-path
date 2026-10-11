---
description: "Beyond basic SQL: LATERAL joins, recursive CTEs with SEARCH and CYCLE, ROLLUP and GROUPING SETS, RANGE window frames, writable CTEs."
---

# 05 · Advanced Querying: LATERAL, Recursive CTEs & Grouping Sets

The [SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/) covers window functions and
basic CTEs. This lesson picks up where it leaves off, with the features that let one PostgreSQL query
replace a loop in application code: `LATERAL` joins for per-row subqueries, recursive CTEs with
built-in ordering and cycle detection, multi-level aggregation in a single pass, date-aware window
frames, and CTEs that modify data.

All examples run against a small, readable dataset on PostgreSQL 18.6:

```sql
CREATE TABLE stores (id int PRIMARY KEY, city text);
CREATE TABLE sales (id int GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
                    store_id int REFERENCES stores, sold_at date, amount numeric(10,2), category text);
INSERT INTO stores VALUES (1,'Pune'),(2,'Delhi'),(3,'Chennai');
INSERT INTO sales (store_id, sold_at, amount, category) VALUES
 (1,'2026-09-01',120,'books'),(1,'2026-09-01',80,'toys'),(1,'2026-09-02',300,'books'),(1,'2026-09-04',50,'toys'),
 (2,'2026-09-01',500,'books'),(2,'2026-09-03',20,'toys'),(2,'2026-09-03',220,'games'),
 (3,'2026-09-02',90,'games');
CREATE INDEX ON sales (store_id, amount DESC);
```

## LATERAL: a subquery that runs for each row

An ordinary subquery in `FROM` cannot refer to other tables in the same `FROM`. A `LATERAL` one can —
it is evaluated once per row of the tables to its left, like a `for` loop:

```sql
SELECT s.city, top.sold_at, top.amount
FROM stores s
CROSS JOIN LATERAL (
  SELECT sold_at, amount FROM sales
  WHERE store_id = s.id
  ORDER BY amount DESC
  LIMIT 2) top
ORDER BY s.city, top.amount DESC;
```

```text
  city   |  sold_at   | amount
---------+------------+--------
 Chennai | 2026-09-02 |  90.00
 Delhi   | 2026-09-01 | 500.00
 Delhi   | 2026-09-03 | 220.00
 Pune    | 2026-09-02 | 300.00
 Pune    | 2026-09-01 | 120.00
```

"Top N per group" is the classic use. With an index on `(store_id, amount DESC)`, each iteration reads
just N index entries, so this stays fast with millions of sales per store — far faster than ranking
every row with `row_number()` and filtering. (For top *1*, `DISTINCT ON` from Level 1 · 08 also works.)

`CROSS JOIN LATERAL` drops outer rows whose subquery returns nothing. To keep them, use
`LEFT JOIN LATERAL ... ON true`:

```sql
INSERT INTO stores VALUES (4, 'Kochi');
SELECT s.city, last_sale.sold_at
FROM stores s
LEFT JOIN LATERAL (SELECT sold_at FROM sales WHERE store_id = s.id
                   ORDER BY sold_at DESC LIMIT 1) last_sale ON true
ORDER BY s.id;
```

```text
  city   |  sold_at
---------+------------
 Pune    | 2026-09-04
 Delhi   | 2026-09-03
 Chennai | 2026-09-02
 Kochi   |
```

Set-returning functions in `FROM` are implicitly lateral, which is why
`FROM events e, jsonb_to_recordset(e.data->'items')` worked in lesson 1.

## Recursive CTEs, with ordering built in

A recursive CTE has a non-recursive seed, `UNION ALL`, and a part that refers to the CTE itself. It
repeats until the recursive part returns no rows. Walking a category tree:

```sql
CREATE TABLE categories (id int PRIMARY KEY, parent_id int REFERENCES categories, name text);
INSERT INTO categories VALUES (1,NULL,'All'),(2,1,'Electronics'),(3,2,'Phones'),(4,2,'Laptops'),
                              (5,3,'Android'),(6,1,'Home'),(7,6,'Kitchen');

WITH RECURSIVE tree AS (
  SELECT id, name, parent_id, 1 AS depth FROM categories WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, c.name, c.parent_id, t.depth + 1
  FROM categories c JOIN tree t ON c.parent_id = t.id
) SEARCH DEPTH FIRST BY name SET ordercol
SELECT repeat('  ', depth - 1) || name AS category, depth FROM tree ORDER BY ordercol;
```

```text
   category    | depth
---------------+-------
 All           |     1
   Electronics |     2
     Laptops   |     3
     Phones    |     3
       Android |     4
   Home        |     2
     Kitchen   |     3
```

The `SEARCH DEPTH FIRST BY name SET ordercol` clause (PostgreSQL 14+) generates a sort key that lists
each node directly under its parent, children ordered by name — exactly the order a tree view needs.
Without it you would build an array path by hand. `SEARCH BREADTH FIRST` orders level by level.

### Cycles

Graph data can loop: ann follows bob, bob follows cy, cy follows ann. A naive "everyone ann can reach"
query would recurse forever (each iteration produces new rows because `hops` keeps increasing). The
`CYCLE` clause detects revisits:

```sql
CREATE TABLE follows (a text, b text);
INSERT INTO follows VALUES ('ann','bob'),('bob','cy'),('cy','ann'),('cy','dee');

WITH RECURSIVE reach AS (
  SELECT b AS person, 1 AS hops FROM follows WHERE a = 'ann'
  UNION ALL
  SELECT f.b, r.hops + 1 FROM follows f JOIN reach r ON f.a = r.person
) CYCLE person SET is_cycle USING path
SELECT person, hops, is_cycle, path FROM reach ORDER BY hops, person;
```

```text
 person | hops | is_cycle |           path
--------+------+----------+--------------------------
 bob    |    1 | f        | {(bob)}
 cy     |    2 | f        | {(bob),(cy)}
 ann    |    3 | f        | {(bob),(cy),(ann)}
 dee    |    3 | f        | {(bob),(cy),(dee)}
 bob    |    4 | t        | {(bob),(cy),(ann),(bob)}
```

When a row's `person` already appears in its `path`, it is emitted with `is_cycle = true` and not
expanded further. Filter `WHERE NOT is_cycle` for the clean result. As a belt-and-braces measure on
any recursive query over user-editable data, also cap depth (`WHERE r.hops < 20`).

## ROLLUP, CUBE and GROUPING SETS

Reports often need subtotals at several levels. Instead of several queries `UNION`ed together, compute
them in one pass:

```sql
SELECT city, category, sum(amount) AS total, GROUPING(city, category) AS lvl
FROM sales JOIN stores ON stores.id = sales.store_id
GROUP BY ROLLUP (city, category)
ORDER BY city NULLS LAST, category NULLS LAST;
```

```text
  city   | category |  total  | lvl
---------+----------+---------+-----
 Chennai | games    |   90.00 |   0
 Chennai |          |   90.00 |   1
 Delhi   | books    |  500.00 |   0
 Delhi   | games    |  220.00 |   0
 Delhi   | toys     |   20.00 |   0
 Delhi   |          |  740.00 |   1
 Pune    | books    |  420.00 |   0
 Pune    | toys     |  130.00 |   0
 Pune    |          |  550.00 |   1
         |          | 1380.00 |   3
```

`ROLLUP (city, category)` produces groups `(city, category)`, `(city)` and `()` — detail, subtotal per
city, grand total. `GROUPING()` returns a bitmask telling you which columns were rolled up, which
matters because a NULL in the output could also be a real NULL in the data. `CUBE (a, b)` produces every
combination; `GROUPING SETS` lets you list exactly the ones you want:

```sql
SELECT city, category, sum(amount)
FROM sales JOIN stores ON stores.id = store_id
GROUP BY GROUPING SETS ((city), (category), ())
ORDER BY 1 NULLS LAST, 2 NULLS LAST;
```

```text
  city   | category |   sum
---------+----------+---------
 Chennai |          |   90.00
 Delhi   |          |  740.00
 Pune    |          |  550.00
         | books    |  920.00
         | games    |  310.00
         | toys     |  150.00
         |          | 1380.00
```

## Window frames over time, not rows

A moving sum "over the last three rows" is wrong when days are missing. Fill the gaps, then use a
`RANGE` frame with an interval:

```sql
WITH daily AS (
  SELECT d::date AS day, coalesce(sum(s.amount), 0) AS total
  FROM generate_series('2026-09-01'::date, '2026-09-05', '1 day') d
  LEFT JOIN sales s ON s.sold_at = d::date
  GROUP BY d)
SELECT day, total,
       sum(total) OVER (ORDER BY day) AS running,
       sum(total) OVER (ORDER BY day
                        RANGE BETWEEN interval '2 days' PRECEDING AND CURRENT ROW) AS last_3_days
FROM daily ORDER BY day;
```

```text
    day     | total  | running | last_3_days
------------+--------+---------+-------------
 2026-09-01 | 700.00 |  700.00 |      700.00
 2026-09-02 | 390.00 | 1090.00 |     1090.00
 2026-09-03 | 240.00 | 1330.00 |     1330.00
 2026-09-04 |  50.00 | 1380.00 |      680.00
 2026-09-05 |      0 | 1380.00 |      290.00
```

`RANGE ... interval '2 days' PRECEDING` includes rows whose `day` is within two days of the current one,
whatever the row count — correct even if you skipped the gap-filling step. `ROWS` frames count rows;
`GROUPS` frames count peer groups. The default frame with `ORDER BY` is
`RANGE UNBOUNDED PRECEDING AND CURRENT ROW`, which is why `running` is a running total.

## Writable CTEs: several changes, one statement

`INSERT`, `UPDATE` and `DELETE` with `RETURNING` can appear inside `WITH`, and their output can feed
another statement. Archiving rows atomically:

```sql
CREATE TABLE sales_archive (LIKE sales);
WITH moved AS (
  DELETE FROM sales WHERE sold_at < '2026-09-02' RETURNING *
)
INSERT INTO sales_archive SELECT * FROM moved;

SELECT (SELECT count(*) FROM sales) AS live, (SELECT count(*) FROM sales_archive) AS archived;
```

```text
 live | archived
------+----------
    5 |        3
```

One statement, one snapshot, no window in which a row exists in both tables or neither. All parts of
a writable CTE see the same snapshot, so one sub-statement cannot see another's changes to the same
table; design them to touch different rows or tables.

## CTEs are inlined unless you say otherwise

Since PostgreSQL 12, a non-recursive, side-effect-free CTE referenced once is inlined into the outer
query, so filters push down into it:

```text
EXPLAIN WITH s AS (SELECT * FROM sales) SELECT * FROM s WHERE store_id = 1;
 Seq Scan on sales
   Filter: (store_id = 1)

EXPLAIN WITH s AS MATERIALIZED (SELECT * FROM sales) SELECT * FROM s WHERE store_id = 1;
 CTE Scan on s
   Filter: (store_id = 1)
   CTE s
     ->  Seq Scan on sales
```

`MATERIALIZED` computes the CTE once, in full, and filters afterwards — worse here, but useful when a
CTE is expensive and referenced several times, or as a deliberate "optimisation fence". `NOT
MATERIALIZED` forces inlining even when the CTE is referenced more than once.

## How It Actually Works

`LATERAL` makes the subquery a function of outer columns, so the planner must execute it as the inner
side of a **nested loop**, passing each outer row's values as parameters — it cannot hash-join it. That
is why an index matching the inner `WHERE` and `ORDER BY` is what makes lateral top-N fast.

A recursive CTE is executed with a **working table**: the seed fills it; each iteration runs the
recursive term against the working table only (the rows produced by the previous iteration), appends
results to the output and makes them the new working table; it stops when an iteration produces
nothing. `UNION` (rather than `UNION ALL`) discards rows already produced, which stops loops only when
rows are exactly identical. `SEARCH` adds a computed column (an array of visited sort keys) and `CYCLE`
adds the path array plus a check on each step — the same thing you would write by hand, generated for
you.

Grouping sets are executed by sorting or hashing once and computing several aggregation levels from
that pass (the plan shows `MixedAggregate` or multiple `Group Key` lines), instead of re-reading the
data per level as a `UNION ALL` of separate `GROUP BY`s would.

## Common mistakes

- `CROSS JOIN LATERAL` where `LEFT JOIN LATERAL ... ON true` was needed, silently dropping rows.
- Recursive queries over user data without cycle detection or a depth limit.
- Confusing a rolled-up NULL with a NULL in the data instead of checking `GROUPING()`.
- `ROWS BETWEEN 2 PRECEDING` for "last three days" on data with gaps.
- Expecting two parts of a writable CTE to see each other's changes.
- Using `MATERIALIZED` out of habit from pre-12 advice.

## Exercise

1. Using `LATERAL`, return each customer's three most recent orders with an order count per customer,
   and create the index that makes it efficient. Check the plan.
2. Model an employee hierarchy and write a recursive query returning each employee with their full
   management chain as a text path ("CEO > VP > Manager"), ordered depth-first.
3. Write a `CUBE` report of sales by city and category and label each row "detail", "city subtotal",
   "category subtotal" or "grand total" using `GROUPING()`.
4. Compute a 7-day moving average of daily revenue that is correct when some days have no sales, with
   and without gap filling. Compare the results.
