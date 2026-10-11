---
description: "PostgreSQL Level 2: MVCC row versions, VACUUM and bloat, pages and TOAST, B-tree and other index types, EXPLAIN ANALYZE, statistics, locking and WAL."
---

# Level 2 · Internals: MVCC, Storage & Indexes

Level 1 taught you to use PostgreSQL well. Level 2 is about understanding why it behaves the way it
does — because every performance problem you will ever debug is explained by one of the mechanisms
here. Why did the table double in size after one `UPDATE`? Why did a one-millisecond `ALTER TABLE`
take the site down? Why is the query fast in psql and slow from the app? Why does the first change
after a checkpoint write 60 times more log than the second?

Each lesson opens up one part of the engine with the tools PostgreSQL gives you for looking inside:
hidden system columns, `pageinspect` and `pgstattuple`, `pg_stats`, `pg_locks`, `pg_waldump`, and
above all `EXPLAIN (ANALYZE, BUFFERS)`. Every output shown was produced on PostgreSQL 18.6; the
project at the end applies all of it to take a realistic workload from 29 to over 22,000
transactions per second.

**Prerequisites:** Level 1, especially transactions and isolation (lesson 7). You will want a
throwaway cluster where you are superuser, because several inspection extensions require it.

## Modules

1. **[MVCC: How Rows Become Visible](01-mvcc.md)** — `xmin`, `xmax`, `ctid`, raw page contents and
   snapshots; why an UPDATE is an insert plus a delete.
2. **[VACUUM, Autovacuum & Bloat](02-vacuum-autovacuum.md)** — reading `VACUUM VERBOSE`, autovacuum
   thresholds, what blocks cleanup, and transaction-ID wraparound.
3. **[Pages, TOAST, Fillfactor & HOT Updates](03-storage-toast-hot.md)** — the 8 kB page, how big
   values are compressed and moved, and updates that skip the indexes.
4. **[B-tree Indexes in Depth](04-btree-indexes.md)** — scan types, column order, skip scan,
   expression, partial and covering indexes, and LIKE prefixes.
5. **[GIN, GiST, BRIN & Hash Indexes](05-other-index-types.md)** — arrays, trigram search,
   nearest-neighbour, and tiny indexes for huge time-ordered tables.
6. **[Reading EXPLAIN ANALYZE](06-explain-analyze.md)** — estimates versus actuals, loops, buffers,
   join methods and disk spills.
7. **[The Planner & Statistics](07-planner-statistics.md)** — `pg_stats`, correlated columns and
   extended statistics, stale stats and generic plans.
8. **[Locks, Deadlocks & SKIP LOCKED](08-locking.md)** — the ALTER TABLE lock queue, deadlock
   detection, job queues and advisory locks.
9. **[WAL, Checkpoints & Durability](09-wal-checkpoints.md)** — LSNs, full-page images, checkpoint
   tuning, `synchronous_commit` and what fsync really guarantees.
10. **[Project — Diagnose a Slow Database](10-project-slow-database.md)** — `pg_stat_statements`,
    `pgbench` and index design applied to a 1 GB help-desk database.

## After Level 2

Level 3 turns to PostgreSQL's advanced features — JSONB, full-text search, PL/pgSQL, triggers,
partitioning and row-level security — with the internals knowledge you now have to judge their costs.
