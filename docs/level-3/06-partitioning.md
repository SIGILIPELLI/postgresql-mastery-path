---
description: "Declarative partitioning: range, list and hash partitions, pruning, unique-key rules, default-partition traps and attach/detach workflows."
---

# 06 · Declarative Partitioning

Partitioning splits one logical table into many physical ones. Queries still use the parent table,
and PostgreSQL routes rows and skips irrelevant partitions automatically. Its real benefits are
operational more than speed: **dropping a month of data becomes a metadata operation instead of a
100-million-row `DELETE`**, VACUUM and index builds work on smaller pieces, and old data can live on
cheaper storage.

It also has rules and traps that bite people who adopt it casually. This lesson shows both, on
PostgreSQL 18.6.

## When to partition

Partition when a table is large *and* you have an access or retention pattern that matches a
partition key — almost always time. Logs, events, metrics and audit trails with "keep 13 months" are
the textbook case. Do not partition a 5 GB table hoping queries get faster: a good index usually wins,
and partitioning adds planning overhead and constraints on keys.

## Range partitioning by month

```sql
CREATE TABLE measurements (
  device_id int NOT NULL,
  taken_at  timestamptz NOT NULL,
  value     double precision
) PARTITION BY RANGE (taken_at);

CREATE TABLE measurements_2026_07 PARTITION OF measurements FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
CREATE TABLE measurements_2026_08 PARTITION OF measurements FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
CREATE TABLE measurements_2026_09 PARTITION OF measurements FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');
```

Ranges are inclusive at `FROM`, exclusive at `TO`, so months tile without gaps or overlaps. A row with
no matching partition is rejected:

```text
INSERT INTO measurements VALUES (1, '2026-10-05', 1.0);
ERROR:  no partition of relation "measurements" found for row
DETAIL:  Partition key of the failing row contains (taken_at) = (2026-10-05 00:00:00+05:30).
```

That is a feature — but it means **you must create future partitions before data arrives**, from a
scheduled job or the `pg_partman` extension (widely used; not run in this course).

After loading 600,000 rows and creating an index on the parent:

```sql
CREATE INDEX ON measurements (device_id, taken_at);
SELECT tableoid::regclass AS partition, count(*) FROM measurements GROUP BY 1 ORDER BY 1;
```

```text
      partition       | count
----------------------+--------
 measurements_2026_07 | 206030
 measurements_2026_08 | 206031
 measurements_2026_09 | 187939
```

An index created on the parent is created on every partition, and on partitions added later.

## Partition pruning

```text
EXPLAIN (ANALYZE) SELECT avg(value) FROM measurements
WHERE taken_at >= '2026-08-10' AND taken_at < '2026-08-11';
 Aggregate (actual time=3.623..3.627 rows=1.00 loops=1)
   ->  Bitmap Heap Scan on measurements_2026_08 measurements (actual time=3.160..3.461 rows=6646.00 loops=1)
         ->  Bitmap Index Scan on measurements_2026_08_device_id_taken_at_idx (...)
               Index Searches: 474
```

Only the August partition appears in the plan: the planner proved the others cannot contain matching
rows. (The `Index Searches: 474` is the skip scan from Level 2 · 04 — the index leads with `device_id`;
an index on `taken_at` would serve this query better.)

Pruning needs a condition **on the partition key, in a form the planner can compare to the bounds**.
Neither of these prunes:

```text
EXPLAIN SELECT avg(value) FROM measurements WHERE device_id = 7;
   ->  Append
         ->  Bitmap Heap Scan on measurements_2026_07 ...
         ->  Bitmap Heap Scan on measurements_2026_08 ...
         ->  Bitmap Heap Scan on measurements_2026_09 ...

EXPLAIN SELECT avg(value) FROM measurements WHERE taken_at::date = '2026-08-10';
               ->  Parallel Append
                     ->  Parallel Seq Scan on measurements_2026_08 ...
                     ->  Parallel Seq Scan on measurements_2026_07 ...
                     ->  Parallel Seq Scan on measurements_2026_09 ...
```

The second is the common mistake: wrapping the key in a cast or function hides it from pruning. Write
ranges on the raw column.

### Pruning at run time

When the value is unknown at plan time — a prepared statement parameter, a subquery, a join — pruning
happens during execution:

```text
PREPARE recent(timestamptz) AS SELECT count(*) FROM measurements WHERE taken_at >= $1;
SET plan_cache_mode = force_generic_plan;
EXPLAIN (ANALYZE) EXECUTE recent('2026-11-15');
               ->  Parallel Append (actual time=0.016..0.016 rows=0.00 loops=3)
                     Subplans Removed: 2
                     ->  Parallel Seq Scan on measurements_2026_11 measurements_1 ...
                     ->  Parallel Seq Scan on measurements_default measurements_2 ...
```

`Subplans Removed: 2` — the generic plan included every partition, and the executor discarded those
it could rule out once `$1` was known.

## Unique constraints must include the partition key

```text
ALTER TABLE measurements ADD PRIMARY KEY (device_id, taken_at);
ALTER TABLE

ALTER TABLE measurements ADD CONSTRAINT m_uniq UNIQUE (device_id);
ERROR:  unique constraint on partitioned table must include all partitioning columns
DETAIL:  UNIQUE constraint on table "measurements" lacks column "taken_at" which is part of the partition key.
```

Each partition has its own index, and PostgreSQL has no global index, so it can only guarantee
uniqueness when the key determines the partition. A table partitioned by time cannot have a primary key
on `id` alone. If you need globally unique IDs, generate them so they cannot collide (an identity
column or UUID) and accept that the database enforces uniqueness only per partition, or include the
partition key in the key — `PRIMARY KEY (id, created_at)`. This is the biggest design consequence of
partitioning; decide it before you migrate.

Foreign keys *referencing* a partitioned table work (since PostgreSQL 12), but must reference that
composite key.

## Retention: drop, do not delete

```text
\timing on
DELETE FROM measurements WHERE taken_at < '2026-08-01';
DELETE 206030
Time: 158.722 ms

DROP TABLE measurements_2026_08;
DROP TABLE
Time: 2.164 ms
```

The `DELETE` removed 206,030 rows in 158 ms on this laptop — and left 206,030 dead tuples for VACUUM
and a WAL record per row. Dropping a partition removed a similar number of rows in 2 ms, with no
VACUUM debt. At a billion rows the difference is hours versus milliseconds.

`DROP TABLE` (or `DETACH PARTITION`) needs a brief `ACCESS EXCLUSIVE` lock on the parent. `DETACH
PARTITION ... CONCURRENTLY` (PostgreSQL 14+) avoids blocking queries on the parent — unless there is a
default partition:

```text
ALTER TABLE measurements DETACH PARTITION measurements_2026_08 CONCURRENTLY;
ERROR:  cannot detach partitions concurrently when a default partition exists
```

## The default partition trap

A `DEFAULT` partition catches rows that fit no other partition:

```sql
CREATE TABLE measurements_default PARTITION OF measurements DEFAULT;
INSERT INTO measurements VALUES (1, '2026-10-05', 1.0);   -- now lands in the default
```

That seems convenient until you create the partition the row *should* have gone to:

```text
CREATE TABLE measurements_2026_10 PARTITION OF measurements FOR VALUES FROM ('2026-10-01') TO ('2026-11-01');
ERROR:  updated partition constraint for default partition "measurements_default" would be violated by some row
```

Adding any partition requires scanning the default partition (under a lock) to prove none of its rows
belong in the new range; if some do, you must move them first. With a large default partition that
scan blocks inserts for its whole duration. Many teams avoid default partitions entirely and instead
create partitions well ahead of time with monitoring that alerts if they run short.

## Attaching a pre-loaded partition

To add a partition holding existing data — a backfill, an archive restore — build it as a normal
table, add a CHECK constraint matching the bounds, then attach:

```sql
CREATE TABLE measurements_2026_11 (LIKE measurements INCLUDING DEFAULTS INCLUDING CONSTRAINTS);
ALTER TABLE measurements_2026_11
  ADD CONSTRAINT m11_range CHECK (taken_at >= '2026-11-01' AND taken_at < '2026-12-01');
-- load data, build indexes ...
ALTER TABLE measurements ATTACH PARTITION measurements_2026_11
  FOR VALUES FROM ('2026-11-01') TO ('2026-12-01');
```

```text
ALTER TABLE
Time: 2.225 ms
```

Because the CHECK constraint already proves every row fits, `ATTACH` skips its validation scan of the
new partition. `ATTACH PARTITION` takes only a `SHARE UPDATE EXCLUSIVE` lock on the parent, so queries
keep running. (The default partition, if any, is still scanned.)

## List and hash partitioning

```sql
CREATE TABLE orders_by_region (id bigint, region text NOT NULL, total numeric) PARTITION BY LIST (region);
CREATE TABLE orders_apac PARTITION OF orders_by_region FOR VALUES IN ('IN', 'JP', 'AU');
CREATE TABLE orders_emea PARTITION OF orders_by_region FOR VALUES IN ('DE', 'FR', 'UK');

CREATE TABLE sessions_h (id bigint, payload text) PARTITION BY HASH (id);
CREATE TABLE sessions_h0 PARTITION OF sessions_h FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_h1 PARTITION OF sessions_h FOR VALUES WITH (MODULUS 4, REMAINDER 1);
```

List partitioning suits a small set of known values (regions, tenants on dedicated tiers). Hash
partitioning spreads rows evenly to make individual partitions smaller, but offers no retention
benefit and prunes only on equality. Create every remainder — with only two of four created:

```text
INSERT INTO sessions_h VALUES (2, 'x');
ERROR:  no partition of relation "sessions_h" found for row
```

Partitions can themselves be partitioned (by region, then month), at the cost of more objects to
manage.

## How many partitions?

Each partition is a table with its own files, indexes, statistics and catalog entries. Planning time
grows with the number of partitions that survive pruning, and every query that cannot prune touches all
of them; locking each partition at execution also costs. Hundreds of partitions are routine;
tens of thousands are not. Size partitions so that each is comfortably manageable (often tens of GB or
less) and the count stays in the hundreds over your retention period.

## How It Actually Works

A partitioned table has no storage of its own; it is a catalog entry (`relkind = 'p'`) with a partition
key in `pg_partitioned_table`, and each partition records its bounds in `pg_class.relpartbound`. On
`INSERT`, tuple routing evaluates the key and binary-searches the sorted bounds to find the target
partition. An `UPDATE` that changes the key moves the row: it is deleted from one partition and
inserted into another.

At planning time, the planner matches `WHERE` clauses on key columns against the bounds to build the
list of surviving partitions, then plans each one separately — which is why partitions can use
different plans and why planning cost scales with survivors. Runtime pruning stores the pruning steps
in the `Append` node and evaluates them once parameter values are known, or even per outer row in a
nested-loop join.

Partition-wise joins and aggregates (`enable_partitionwise_join`, `enable_partitionwise_aggregate`,
off by default because they increase planning time) let the planner join or aggregate matching
partitions pair by pair instead of the whole tables.

## Common mistakes

- Partitioning a table that only needed an index.
- Forgetting to create next month's partition.
- Wrapping the partition key in casts or functions in queries, defeating pruning.
- Designing around a primary key on `id` alone, then discovering it cannot exist.
- Relying on a default partition and then being unable to add partitions without long locks.
- `DELETE`-based retention on a partitioned table instead of dropping partitions.

## Exercise

1. Create a `page_views` table partitioned by day for the last 14 days plus 7 future days, load a few
   million rows with `generate_series`, and verify with `EXPLAIN` that a one-day query touches one
   partition.
2. Write a function that creates any missing partitions for the next N days, and schedule it (cron,
   or the `pg_cron` extension if you have it).
3. Implement 7-day retention by detaching and dropping old partitions with `DETACH ... CONCURRENTLY`.
   What must be true about the table for that to work?
4. Compare `DELETE` of one day versus dropping its partition: time, WAL generated (`pg_wal_lsn_diff`)
   and dead tuples left behind.
