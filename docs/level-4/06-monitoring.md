---
description: "Monitor live PostgreSQL: pg_stat_activity and wait events, progress views, pg_stat_database, pg_stat_io, auto_explain and alerts."
---

# 06 · Monitoring a Running Database

Previous lessons used PostgreSQL's statistics views to investigate one problem at a time. Operating a
database means watching them continuously, so you notice the long transaction before it bloats every
table, the inactive slot before the disk fills, and the slow query before users do. Everything you need
is built in; dashboards and agents (Prometheus exporters, Datadog, pganalyze and others — none run here)
mostly query these same views.

Output is from PostgreSQL 18.6, with a few sessions deliberately misbehaving.

## What is happening right now: pg_stat_activity

Four sessions were set up: one updated a row and then sat idle inside its transaction; a second tried to
update the same row; a third ran a long `pg_sleep`; a fourth started `CREATE INDEX CONCURRENTLY`.

```sql
SELECT pid, state, wait_event_type || ':' || wait_event AS waiting_on,
       date_trunc('second', now() - xact_start) AS xact_age, left(query, 45) AS query
FROM pg_stat_activity
WHERE backend_type = 'client backend' AND pid <> pg_backend_pid()
ORDER BY xact_start NULLS LAST;
```

```text
(67055, 'idle in transaction', 'Client:ClientRead',  1 s, 'UPDATE orders SET total = total WHERE id = 1')
(67056, 'active',              'Lock:transactionid', 1 s, 'UPDATE orders SET total = total WHERE id = 1')
(67057, 'active',              'Timeout:PgSleep',    1 s, 'SELECT pg_sleep(30)')
(67058, 'active',              'Lock:virtualxid',    1 s, 'CREATE INDEX CONCURRENTLY orders_total_idx ON')
```

How to read it:

- **state** — `active` (running), `idle` (connected, nothing to do), `idle in transaction` (inside an
  open transaction, doing nothing — the dangerous one), `idle in transaction (aborted)`.
- **query** — for idle sessions, the *last* query run, not a current one.
- **wait_event_type / wait_event** — what an active session is blocked on. `Lock:transactionid` is a
  row lock held by another transaction (Level 2 · 08); `Client:ClientRead` means waiting for the client
  to send something; `IO:DataFileRead` is disk reads; `LWLock:*` is internal contention. Sampling wait
  events every second is a cheap profiler for the whole server.
- **xact_age** — long-open transactions block VACUUM cleanup cluster-wide (Level 2 · 02).

The two sessions waiting on locks were both blocked by the first one, which was idle: nothing would ever
move until it committed or was killed. `pg_blocking_pids(pid)` shows who blocks whom;
`pg_cancel_backend(pid)` cancels a query, `pg_terminate_backend(pid)` ends the session.

## Progress views

Long maintenance operations report progress:

```sql
SELECT phase, blocks_done, blocks_total, lockers_total, lockers_done FROM pg_stat_progress_create_index;
```

```text
[('waiting for writers before build', 0, 0, 2, 0)]
-- after the idle transaction was rolled back:
[('waiting for old snapshots',)]
```

This is worth knowing: `CREATE INDEX CONCURRENTLY` was stuck in its first phase waiting for the two
transactions writing to the table — including the *idle* one — and then in a later phase waiting for
the 30-second `pg_sleep` query's snapshot to go away, even though that query touched nothing related.
A concurrent index build on a busy system can wait a long time on unrelated long transactions; the
progress view tells you which phase it is in. Similar views exist for VACUUM, ANALYZE, CLUSTER, COPY and
base backups (`pg_stat_progress_*`).

## Database-level counters: pg_stat_database

```sql
SELECT datname, numbackends, xact_commit, xact_rollback,
       round(100.0 * blks_hit / nullif(blks_hit + blks_read, 0), 2) AS cache_hit_pct,
       temp_files, pg_size_pretty(temp_bytes) AS temp, deadlocks, conflicts
FROM pg_stat_database WHERE datname IN ('perf', 'desk', 'bench', 'lab') ORDER BY datname;
```

```text
 datname | numbackends | xact_commit | xact_rollback | cache_hit_pct | temp_files |  temp   | deadlocks | conflicts
---------+-------------+-------------+---------------+---------------+------------+---------+-----------+-----------
 bench   |           0 |     8741911 |             1 |         89.56 |          4 | 96 MB   |         0 |         0
 desk    |           0 |      680772 |             1 |         77.23 |         16 | 274 MB  |         0 |         0
 lab     |           0 |      112615 |            66 |         99.93 |          2 | 1968 kB |         2 |         0
 perf    |           1 |         381 |             2 |         99.06 |         42 | 523 MB  |         0 |         0
```

These are **cumulative counters** since the last reset — useful as rates (sample every minute, graph
the difference), nearly meaningless as single values. The `desk` cache hit rate of 77% reflects the
Level 2 project's sequential scans of a 1 GB database, and `perf`'s 42 temp files come from the
deliberate `work_mem` spills in Level 2 · 06. On a production OLTP database a cache hit rate in the high
90s is typical; a sudden drop, a rise in `temp_bytes`, or any `deadlocks` are worth investigating. A
"blocks read" here can still be served from the OS cache (Level 4 · 01), so this is not a disk-read rate.

## I/O by activity: pg_stat_io

PostgreSQL 16 added `pg_stat_io`, breaking I/O down by who did it and why:

```text
   backend_type    |  object  |  context  |  reads   | writes  | extends |   hits    | evictions
-------------------+----------+-----------+----------+---------+---------+-----------+-----------
 client backend    | relation | normal    | 10661416 | 1895817 |  275867 | 180267125 |  12238902
 background worker | relation | normal    |   356642 |    6296 |       0 |   1328142 |   2634276
 client backend    | relation | vacuum    |    49415 |   96946 |       0 |    147442 |      1304
 client backend    | relation | bulkwrite |        0 |  103618 |    2094 |    106449 |      7948
 autovacuum worker | relation | vacuum    |    17132 |   65734 |       0 |    100913 |      2961
 autovacuum worker | relation | normal    |     3330 |     352 |      75 |    236474 |      2776
```

One line stands out: **client backends wrote 1.9 million buffers themselves**. Ideally the background
writer and checkpointer write dirty pages and client backends only read; backends writing means they
had to clean a buffer before they could use it, adding latency to queries. On this lab that is the
result of bulk loads into a small `shared_buffers`; on a production server it is a sign to look at
`shared_buffers`, `bgwriter_lru_maxpages` and checkpoint settings. The `vacuum` and `bulkwrite`
contexts show I/O done through the ring buffers mentioned in lesson 1.

## Slow queries: logs, pg_stat_statements and auto_explain

Three complementary tools:

- `log_min_duration_statement = '250ms'` logs every slow statement with its duration and parameters.
- `pg_stat_statements` (Level 2 · 10) aggregates every statement shape — the best "where does the time
  go" view.
- **`auto_explain`** logs the *plan* of slow statements as they actually ran, which matters when the
  problem only happens with production data or parameters:

```sql
LOAD 'auto_explain';                         -- or add to shared_preload_libraries / session_preload_libraries
SET auto_explain.log_min_duration = '20ms';
SET auto_explain.log_analyze = on;
SET auto_explain.log_buffers = on;
SELECT count(*) FROM orders WHERE email LIKE '%31337%';
```

In the server log:

```text
LOG:  duration: 62.420 ms  plan:
	Query Text: SELECT count(*) FROM orders WHERE email LIKE '%31337%';
	Finalize Aggregate  (cost=17119.65..17119.66 rows=1 width=8) (actual time=61.016..62.408 rows=1.00 loops=1)
	  Buffers: shared hit=10911
	  ->  Gather  (cost=17119.44..17119.65 rows=2 width=8) (actual time=60.933..62.404 rows=3.00 loops=1)
	        Workers Planned: 2
	        Workers Launched: 2
	        ->  Partial Aggregate  (...)
	              ->  Parallel Seq Scan on orders  (cost=0.00..16119.33 rows=42 width=0) (actual time=9.310..54.225 rows=6.67 loops=3)
	                    Filter: (email ~~ '%31337%'::text)
	                    Rows Removed by Filter: 333327
```

`log_analyze` adds timing overhead to every statement (not just slow ones) because it must instrument
them all; in production, many teams enable it with `auto_explain.sample_rate` below 1, or use
`log_timing = off`.

## What to alert on

A practical starting list — each threshold depends on your system, so tune after a few weeks of data:

| Signal | Query / source | Why |
|--------|----------------|-----|
| Server reachable, accepting writes | `SELECT 1`, `pg_is_in_recovery()` | obvious, and catches unexpected failovers |
| Connections near `max_connections` | `count(*)` from `pg_stat_activity` | new connections will be refused |
| Oldest transaction / idle-in-transaction age | `now() - xact_start` | blocks VACUUM, holds locks |
| Sessions waiting on locks for long | `pg_blocking_pids` | outages start this way |
| Replication lag (bytes and time) | `pg_stat_replication` (lesson 3) | failover data loss, stale reads |
| Inactive slots, retained WAL | `pg_replication_slots` | disk full |
| Archiver failures | `pg_stat_archiver.failed_count` | broken PITR (lesson 5) |
| Transaction ID age | `age(datfrozenxid)` (Level 2 · 02) | wraparound shutdown |
| Disk free for data and WAL | OS metrics | the most common outage cause of all |
| Error rates, deadlocks, temp spills | `pg_stat_database` rates, logs | regressions |
| Top statements by total time | `pg_stat_statements` | capacity and regressions |

And keep the logs: `log_line_prefix` with time, PID, user and database; `log_lock_waits`;
`log_autovacuum_min_duration`; `log_temp_files`; `log_checkpoints`. The capstone (lesson 10) packages a
health-check script from these queries.

## How It Actually Works

Each backend accumulates statistics (rows read, blocks hit, I/O counts, function calls) in local memory
and periodically flushes them into **shared-memory** statistics entries — since PostgreSQL 15 the
cumulative statistics system lives entirely in shared memory and is written to disk only at clean
shutdown, which is why counters reset after a crash. `pg_stat_*` views read those entries; there is
deliberately a short delay before a backend's numbers become visible (the
`pg_stat_force_next_flush()` you saw in Level 2 · 03 bypasses it).

`pg_stat_activity` is different: it reads each backend's live status slot in shared memory, updated as
the backend changes state, so it is current to the moment. Wait events are a single field each backend
sets before blocking and clears after — cheap enough to leave on always, which is why sampling them works
so well.

## Common mistakes

- Treating cumulative counters as current values.
- Monitoring only CPU and memory, not transaction age, slots, archiver and wraparound.
- Killing the blocked sessions instead of the one blocking them.
- `auto_explain.log_analyze = on` with `log_min_duration = 0` on a busy server.
- No disk-space alerts on the WAL volume.

## Exercise

1. Write a `health_check.sql` that returns one row per problem found: idle-in-transaction sessions older
   than 5 minutes, lock waits over 30 s, inactive slots, archiver failures in the last hour, and
   databases with `age(datfrozenxid)` above 500 million.
2. Sample `wait_event_type, wait_event` from `pg_stat_activity` every second for a minute while running
   `pgbench`, and summarise which waits dominate.
3. Enable `auto_explain` for one role with `ALTER ROLE ... SET session_preload_libraries = 'auto_explain'`
   and find a slow plan in the log.
4. Reset `pg_stat_io` (`SELECT pg_stat_reset_shared('io')`), run a bulk load, and explain the counts in
   each context afterwards.
