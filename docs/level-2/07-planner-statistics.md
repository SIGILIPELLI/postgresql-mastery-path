---
description: "How the PostgreSQL planner estimates rows: pg_stats, MCVs, histograms, correlated columns, CREATE STATISTICS, stale stats and generic plans."
---

# 07 · The Planner & Statistics

PostgreSQL's planner is cost-based. For every query it considers many possible plans — scan methods,
join orders, join methods — estimates the cost of each, and runs the cheapest. The cost model is
good. What usually goes wrong is its *inputs*: the estimated number of rows flowing through each
step. A plan built for 100 rows that meets 100,000 is how a 5 ms query becomes a 50-second one.

This lesson shows where estimates come from, the classic ways they go wrong, and how to fix each.
Outputs are from PostgreSQL 18.6 in the `perf` database.

## Where estimates come from: pg_stats

`ANALYZE` (run by autovacuum, or by hand) samples each table — 300 × `default_statistics_target`
rows, so 30,000 with the default of 100 — and stores per-column statistics, readable in `pg_stats`:

```text
SELECT attname, n_distinct, most_common_vals, most_common_freqs, correlation
FROM pg_stats WHERE tablename = 'orders' AND attname IN ('status','customer_id');
   attname   | n_distinct |        most_common_vals         |           most_common_freqs           | correlation
-------------+------------+---------------------------------+---------------------------------------+--------------
 customer_id |      47965 | {17198,18719,34253}             | {0.0002,0.0002,0.0002}                | -0.002201437
 status      |          4 | {paid,shipped,refunded,pending} | {0.4379,0.37676665,0.123333335,0.062} |    0.3514488
```

- **n_distinct** — estimated number of distinct values (negative numbers mean "a fraction of the
  row count").
- **most_common_vals / most_common_freqs (MCV)** — the most frequent values and their share. For
  `status = 'pending'` the planner simply looks up 0.062 × 1,000,000 ≈ 62,000 rows.
- **histogram_bounds** — for values not in the MCV list, boundaries that split the remaining values
  into equal-population buckets. Ranges such as `total < 10` are estimated by counting how many
  buckets fall below 10:

```text
 buckets |   first_bounds
---------+-------------------
     101 | {0.06,5.27,10.36}
```

- **correlation** — how closely physical row order follows the column's sort order (1 = perfectly).
  It affects the cost of index scans (and, as lesson 5 showed, whether BRIN works).

For an equality on a value outside the MCV list, the planner assumes the remaining rows are spread
evenly over the remaining distinct values.

## Problem 1: correlated columns

The planner assumes conditions on different columns are **independent**, and multiplies their
selectivities. Real data is rarely independent:

```sql
CREATE TABLE addresses (id int PRIMARY KEY, city text, country text);
-- 300,000 rows over 10 cities; every city is in exactly one country
```

```text
EXPLAIN (ANALYZE) SELECT * FROM addresses WHERE city = 'Pune' AND country = 'IN';
 Seq Scan on addresses  (cost=0.00..6122.00 rows=9065 width=13) (actual time=0.007..18.199 rows=30000.00 loops=1)
   Filter: ((city = 'Pune'::text) AND (country = 'IN'::text))
```

The planner computed P(city = Pune) ≈ 0.1 × P(country = IN) ≈ 0.3 → about 9,000 rows. But every
Pune address is in India, so the true answer is 30,000 — off by more than 3×. Combine three or four
such conditions, or feed the estimate into a join, and the error compounds.

**Extended statistics** tell the planner about relationships between columns:

```sql
CREATE STATISTICS addresses_city_country (dependencies, ndistinct, mcv)
  ON city, country FROM addresses;
ANALYZE addresses;
```

```text
 Seq Scan on addresses  (cost=0.00..6122.00 rows=29490 width=13) (actual time=0.005..18.374 rows=30000.00 loops=1)
```

Now 29,490 versus 30,000. The three kinds:

- `dependencies` — "city functionally determines country", used for equality conditions.
- `ndistinct` — the number of distinct *combinations*, used for `GROUP BY city, country` estimates.
  Without it, the planner multiplies distinct counts and can expect thousands of groups where there
  are ten.
- `mcv` — a multi-column most-common-values list, the most precise for equality and IN lists.

Since PostgreSQL 14 you can also create statistics on **expressions**, e.g.
`CREATE STATISTICS ... ON (lower(email)) FROM users` — useful when you filter on an expression you
have no index on.

## Problem 2: expressions the planner cannot see into

```text
EXPLAIN (ANALYZE) SELECT * FROM addresses WHERE lower(city) = 'pune';
 Gather  (cost=1000.00..5419.06 rows=1500 width=13) (actual time=0.101..23.420 rows=30000.00 loops=1)
   ->  Parallel Seq Scan on addresses  (...)
         Filter: (lower(city) = 'pune'::text)
```

With no statistics on `lower(city)`, PostgreSQL falls back to a hard-coded default selectivity for
equality (0.5%: 1,500 of 300,000). Actual: 30,000 — 20× more. Fixes: an expression index (which
gets its own statistics from ANALYZE), expression statistics as above, or, better, store the data in
the form you query.

## Problem 3: stale statistics

```sql
CREATE TABLE fresh (id int, v int) WITH (autovacuum_enabled = false);
INSERT INTO fresh SELECT g, g % 10 FROM generate_series(1, 1000000) g;
EXPLAIN SELECT * FROM fresh WHERE v = 3;
```

```text
 Gather  (cost=1000.00..11133.59 rows=5000 width=8)
   Workers Planned: 2
   ->  Parallel Seq Scan on fresh  (cost=0.00..9633.59 rows=2083 width=8)
```

The table has never been analysed; the planner knows its size from the page count but nothing about
`v`, so it guesses 0.5% again: 5,000 rows. After `ANALYZE fresh`:

```text
 Seq Scan on fresh  (cost=0.00..16925.00 rows=103067 width=8)
```

The true answer is 100,000. Autovacuum would have analysed this table within a minute or so — but
in a migration or ETL job that loads a table and immediately queries it, there is no "minute or
so". **Run `ANALYZE` after bulk loads**, and after restoring a dump (lesson Level 1 · 09).

Statistics also go stale on tables whose *distribution* changes faster than autovacuum's analyze
threshold (10% of rows changed by default). The classic case: a `created_at` column queried for
"the last hour", on a table where the newest hour is always beyond the end of the histogram. For
such tables lower `autovacuum_analyze_scale_factor` per table.

## Raising the statistics target

For columns with many distinct values and a skewed distribution, 100 MCV entries and 100 histogram
buckets may not capture enough. Raise the target for that column only:

```sql
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 1000;
ANALYZE orders;
```

ANALYZE then samples more rows and stores larger lists, at the cost of slower ANALYZE and slightly
slower planning for queries that use the column.

## Problem 4: generic plans for prepared statements

Drivers often use prepared statements: plan once, execute many times with different parameters.
PostgreSQL plans the first five executions with the actual values ("custom plans"), then compares
their average cost with a **generic plan** made without knowing the values. If the generic plan is
not more expensive, it switches to it for good.

With skewed data that can backfire. A table where 99.9% of rows have `kind = 'common'` and 0.1%
`'rare'`:

```sql
PREPARE by_kind2(text) AS SELECT count(payload) FROM skew WHERE kind = $1;
EXECUTE by_kind2('common');   -- five times
EXPLAIN (ANALYZE, COSTS OFF) EXECUTE by_kind2('rare');
```

```text
 Finalize Aggregate (actual time=2.030..2.694 rows=1.00 loops=1)
   ->  Gather (actual time=0.342..2.693 rows=2.00 loops=1)
         Workers Planned: 1
         Workers Launched: 1
         ->  Partial Aggregate (actual time=0.141..0.141 rows=1.00 loops=2)
               ->  Parallel Index Scan using skew_kind_idx on skew (actual time=0.013..0.128 rows=250.00 loops=2)
                     Index Cond: (kind = $1)
 Execution Time: 2.711 ms
```

`Index Cond: (kind = $1)` — the parameter, not the value — is the sign of a generic plan, and
`pg_prepared_statements` confirms it (`generic_plans = 1, custom_plans = 5`). Forcing custom
planning:

```sql
SET plan_cache_mode = force_custom_plan;
```

```text
 Aggregate (actual time=0.293..0.293 rows=1.00 loops=1)
   ->  Index Scan using skew_kind_idx on skew (actual time=0.008..0.269 rows=500.00 loops=1)
         Index Cond: (kind = 'rare'::text)
 Execution Time: 0.299 ms
```

Here the difference is milliseconds; on a big table a generic plan can mean a sequential scan for a
value that matches ten rows. Symptoms: "the query is fast in psql and slow from the application".
Fixes, from narrowest to broadest: `plan_cache_mode = force_custom_plan` for the role or session,
avoiding server-side prepared statements for that query in the driver, or restructuring the query so
the skewed value is a literal (for example a partial index for the rare value).

## Cost settings

The planner turns row estimates into costs using constants:

```text
         name         | setting
----------------------+---------
 cpu_tuple_cost       | 0.01
 effective_cache_size | 524288        (8 kB pages = 4 GB)
 random_page_cost     | 4
 seq_page_cost        | 1
```

`random_page_cost = 4` says a random page read costs four times a sequential one — a sensible
assumption for spinning disks. On SSDs, or when the working set is cached, `1.1`–`2` is common and
makes index scans look appropriately cheaper. `effective_cache_size` tells the planner how much data
is likely to be cached (shared buffers plus OS cache); it allocates nothing. Level 4 · 01 sets both
for a real server.

## Diagnosing with enable_* switches

`SET enable_seqscan = off` (and `enable_nestloop`, `enable_hashjoin`, `enable_sort`, …) makes the
planner treat a method as extremely expensive. It is a **diagnostic** tool: if disabling a nested
loop makes the query 100× faster, you have learned the planner's cost estimate for the alternative
was wrong — usually because of a row misestimate you should fix at the source. Never leave these off
in production configuration. PostgreSQL 18's `EXPLAIN` marks nodes the planner was forced to use
against a disabled setting with `Disabled: true`.

## How It Actually Works

Planning happens in stages. The query is parsed and rewritten (views expanded), then the planner
estimates the **selectivity** of each restriction clause using the column statistics — MCV lookup,
histogram interpolation, or hard-coded defaults when nothing is known (0.5% for equality, 33% for
inequalities). Multiplying by the table's row estimate (`reltuples` scaled by its current page count)
gives each scan's output rows.

For joins, it estimates join selectivity from both sides' statistics (n_distinct and MCVs on the join
keys), then searches join orders: exhaustively with dynamic programming up to
`join_collapse_limit` / `from_collapse_limit` tables (8 by default), and with the genetic query
optimiser above `geqo_threshold` (12). Each candidate path has a startup and total cost computed
from page costs, per-tuple CPU costs and the estimated rows; the cheapest path at each level is kept,
with extra paths retained if they produce useful sort orders.

Because errors multiply through joins, an estimate that is 10× too low at the bottom of a five-way
join can be 1,000× wrong at the top, turning a sensible hash join into a nested loop that runs a
million times.

## Common mistakes

- Not running `ANALYZE` after bulk loads or restores.
- Assuming the planner knows two columns are related.
- Filtering on expressions with neither an index nor expression statistics.
- Disabling planner methods globally to "fix" one query.
- Leaving `random_page_cost = 4` on all-SSD systems without testing.
- Benchmarking a query in psql with literals and concluding "it's fast", when the application runs
  it as a prepared statement with a generic plan.

## Exercise

1. Find a pair of correlated columns in your own data (city/postcode, product/category). Show the
   misestimate, add extended statistics, and show the improvement.
2. Run `SELECT city, country, count(*) ... GROUP BY city, country` on `addresses` before and after
   creating `ndistinct` statistics and compare the estimated group count.
3. Bulk-load a table with autovacuum disabled and write a join against it. Compare the plan before
   and after `ANALYZE`.
4. Reproduce the generic-plan problem on a table of your own where one value is extremely common.
   Then fix it with a partial index for the rare value, without changing `plan_cache_mode`.
