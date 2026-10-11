---
description: "What VACUUM really does, reading VACUUM VERBOSE, autovacuum thresholds, why long transactions cause bloat, and watching XID wraparound."
---

# 02 · VACUUM, Autovacuum & Bloat

The previous lesson showed that every UPDATE and DELETE leaves an old row version behind. Something
has to clean those up, and that something is **VACUUM**. It also does a second, less famous job
that is a matter of survival rather than tidiness: **freezing** old rows so that 32-bit transaction
IDs can wrap around safely.

Most production PostgreSQL problems that look mysterious — a table that keeps growing, queries that
slowly get slower, a database that suddenly refuses writes — trace back to VACUUM not being able to
do its job. This lesson shows what it does, how to read its output, and what stops it.

All output is from PostgreSQL 18.6.

## Dead tuples in numbers

A test table with autovacuum turned off, so we control everything:

```sql
CREATE EXTENSION pgstattuple;
CREATE TABLE vac (id int PRIMARY KEY, payload text) WITH (autovacuum_enabled = false);
INSERT INTO vac SELECT g, repeat('x', 100) FROM generate_series(1, 100000) g;
SELECT pg_size_pretty(pg_relation_size('vac'));          -- 13 MB

UPDATE vac SET payload = repeat('y', 100);
SELECT pg_size_pretty(pg_relation_size('vac'));          -- 27 MB
```

Updating every row doubled the table. `pgstattuple` scans it and reports exactly what is inside:

```text
SELECT tuple_count, dead_tuple_count, round(dead_tuple_percent) AS dead_pct,
       round(free_percent) AS free_pct FROM pgstattuple('vac');
 tuple_count | dead_tuple_count | dead_pct | free_pct
-------------+------------------+----------+----------
      100000 |           100000 |       46 |        1
```

Half the table is dead row versions.

## What plain VACUUM does

```sql
VACUUM (VERBOSE) vac;
```

```text
INFO:  vacuuming "lab.public.vac"
INFO:  finished vacuuming "lab.public.vac": index scans: 1
pages: 0 removed, 3449 remain, 3449 scanned (100.00% of total), 0 eagerly scanned
tuples: 100000 removed, 100000 remain, 0 are dead but not yet removable
removable cutoff: 101341, which was 0 XIDs old when operation ended
new relfrozenxid: 101340, which is 2 XIDs ahead of previous value
frozen: 0 pages from table (0.00% of total) had 0 tuples frozen
visibility map: 3449 pages set all-visible, 1724 pages set all-frozen (0 were all-visible)
index scan needed: 1725 pages from table (50.01% of total) had 100000 dead item identifiers removed
index "vac_pkey": pages: 551 in total, 0 newly deleted, 0 currently deleted, 0 reusable
...
WAL usage: 7450 records, 3 full page images, 1310669 bytes, 0 buffers full
system usage: CPU: user: 0.02 s, system: 0.00 s, elapsed: 0.02 s
```

Line by line:

- **tuples: 100000 removed** — the dead versions are gone.
- **0 are dead but not yet removable** — nothing was blocked (keep an eye on this number).
- **removable cutoff** — the oldest transaction ID any running snapshot could still need. Versions
  deleted before it are garbage; versions deleted after it might still be visible to someone.
- **index scans: 1** — VACUUM also removed the index entries pointing at dead tuples.
- **visibility map** — pages where every tuple is visible to everyone were marked *all-visible*
  (helps index-only scans, lesson 4, and lets future vacuums skip them) and some *all-frozen*.
- **new relfrozenxid** — the table's oldest unfrozen transaction ID moved forward (more below).

Now the size:

```text
SELECT pg_size_pretty(pg_relation_size('vac'));   -- 27 MB
SELECT tuple_count, dead_tuple_count, round(free_percent) FROM pgstattuple('vac');
 tuple_count | dead_tuple_count | free_pct
-------------+------------------+----------
      100000 |                0 |       50
```

**Plain VACUUM does not shrink the file.** It turns dead space into free space *inside* the pages
(recorded in the free space map) for future inserts and updates to reuse. Watch that happen:

```sql
UPDATE vac SET payload = repeat('z', 100);
SELECT pg_size_pretty(pg_relation_size('vac'));   -- still 27 MB
```

The second full-table update reused the space instead of growing the table. That is the healthy
steady state: a table updated regularly settles at a size with some free space, and stays there.
(Plain VACUUM can only return space to the OS when the *last* pages of the file become completely
empty, by truncating them.)

## VACUUM FULL: shrinking, at a price

```sql
VACUUM FULL vac;
SELECT pg_size_pretty(pg_relation_size('vac'));   -- 13 MB
```

`VACUUM FULL` writes a brand-new, compact copy of the table and its indexes, then swaps it in. It
needs an **ACCESS EXCLUSIVE lock for the whole duration** — no reads, no writes — and temporarily
needs disk space for the second copy. On a large production table that is an outage. Use it only
when a table has bloated far beyond its steady state (for example after deleting 90% of it for
good), and consider the `pg_repack` extension, which rebuilds with only brief locks (Level 3 · 08
mentions it; not run in this course).

## Autovacuum: when does it run?

You should almost never need to run VACUUM by hand. The autovacuum launcher wakes every
`autovacuum_naptime` and starts workers for tables that cross a threshold:

```text
                 name                  |  setting
---------------------------------------+------------
 autovacuum_analyze_scale_factor       | 0.1
 autovacuum_freeze_max_age             | 200000000
 autovacuum_max_workers                | 3
 autovacuum_naptime                    | 60
 autovacuum_vacuum_cost_limit          | -1
 autovacuum_vacuum_insert_scale_factor | 0.2
 autovacuum_vacuum_insert_threshold    | 1000
 autovacuum_vacuum_max_threshold       | 100000000
 autovacuum_vacuum_scale_factor        | 0.2
 autovacuum_vacuum_threshold           | 50
 autovacuum_worker_slots               | 16
 vacuum_failsafe_age                   | 1600000000
```

A table is vacuumed when

```text
dead tuples > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor × live tuples
```

capped (new in PostgreSQL 18) at `autovacuum_vacuum_max_threshold`. With defaults, a 1,000-row table
is vacuumed after 250 dead rows, and a 100-million-row table after about 20 million. That second
number is why big, busy tables often need **per-table settings**:

```sql
ALTER TABLE events SET (autovacuum_vacuum_scale_factor = 0.01,
                        autovacuum_vacuum_cost_limit   = 2000);
```

Similar formulas trigger vacuums for insert-only tables (`*_insert_*`, so that they get frozen and
marked all-visible) and `ANALYZE` (statistics, lesson 7).

PostgreSQL 18 also separated `autovacuum_worker_slots` (reserved at start-up, needs a restart) from
`autovacuum_max_workers` (how many may run, changeable with a reload).

### Throttling

Autovacuum deliberately runs slowly so it does not swamp the disks: it accumulates "cost" per page
read or dirtied and sleeps for `autovacuum_vacuum_cost_delay` (2 ms) each time it reaches the cost
limit (default 200, shared across workers). On modern SSDs those defaults are conservative. If
`n_dead_tup` keeps climbing on a busy table even though autovacuum keeps running, raise the cost
limit before anything else.

Check what autovacuum has been doing:

```sql
SELECT relname, n_live_tup, n_dead_tup, last_vacuum, last_autovacuum,
       autovacuum_count, last_autoanalyze
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;
```

## The number-one bloat cause: something holding an old snapshot

Here a session opens a Repeatable Read transaction, reads one row, and then sits idle — exactly
what an application does when it forgets to commit, or what a person does when they leave a `BEGIN`
in a psql window over lunch. Meanwhile another session updates 10,000 rows and runs VACUUM:

```text
tuples: 0 removed, 110000 remain, 10000 are dead but not yet removable
removable cutoff: 101343, which was 1 XIDs old when operation ended
```

Nothing was removed. The idle session's snapshot might still need to see the old versions, so
VACUUM must keep them — **in every table in the database**, not just the ones that session touched.
The culprit is visible in `pg_stat_activity`:

```sql
SELECT pid, state, now() - xact_start AS xact_age, backend_xmin
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY age(backend_xmin) DESC;
```

```text
[(46577, 'idle in transaction', True, '101343')]
```

Once that transaction ended, the same VACUUM removed everything:

```text
tuples: 10000 removed, 97770 remain, 0 are dead but not yet removable
```

(The "remain" figure is an estimate, because VACUUM skipped pages the visibility map already marked
all-visible.)

Things that hold back the removable cutoff:

1. Long-running or **idle-in-transaction** sessions. Set `idle_in_transaction_session_timeout`
   (for example `5min`) so forgotten transactions are killed.
2. Long-running queries on the primary, including reporting queries and `pg_dump`.
3. **Replication slots** that are not being consumed (Level 4 · 03/04) — `pg_replication_slots.xmin`
   and `catalog_xmin`.
4. Standby queries with `hot_standby_feedback = on`.
5. Prepared transactions (`pg_prepared_xacts`) that were never committed or rolled back.

## Freezing and wraparound

Because tuple transaction IDs are 32 bits (lesson 1), VACUUM must eventually mark every old row as
**frozen** — visible to all, regardless of its `xmin`. Each table tracks its oldest unfrozen ID in
`pg_class.relfrozenxid`, and each database the minimum in `pg_database.datfrozenxid`:

```sql
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;
```

```text
    datname    |  age
---------------+--------
 postgres      | 100600
 ...
```

`age()` is "how many transactions ago". The thresholds that matter:

| Age | What happens |
|-----|--------------|
| `autovacuum_freeze_max_age` (200 million) | autovacuum forces an **aggressive** anti-wraparound vacuum of the table, even if autovacuum is disabled for it |
| `vacuum_failsafe_age` (1.6 billion) | VACUUM drops its cost throttling and skips index cleanup to finish freezing as fast as possible |
| about 2 billion minus 3 million | the server **refuses to assign new transaction IDs** — no writes — until a VACUUM completes |

The last row is the outage you read about in post-mortems. It is entirely preventable: monitor
`age(datfrozenxid)` and the per-table `age(relfrozenxid)`, alert at, say, 500 million, and fix
whatever is blocking vacuum long before it matters. A forgotten replication slot or a transaction
left open for days is the usual cause.

```sql
SELECT c.oid::regclass AS table_name, age(c.relfrozenxid) AS xid_age,
       pg_size_pretty(pg_table_size(c.oid)) AS size
FROM pg_class c
WHERE c.relkind IN ('r', 'm', 't')
ORDER BY age(c.relfrozenxid) DESC
LIMIT 10;
```

## How It Actually Works

A lazy (plain) VACUUM of a table runs in phases:

1. **Scan heap.** Read pages that the visibility map does not mark as all-visible. On each page,
   *prune*: dead tuples whose `xmax` is older than the removable cutoff are removed, their space
   compacted, and their line pointers set to **dead** (they cannot be reused yet, because index
   entries still point to them). Collect the dead line-pointer IDs in memory (`maintenance_work_mem`
   or `autovacuum_work_mem` bounds this; since PostgreSQL 17 it uses a compact radix-tree structure).
   Freeze tuples that are old enough.
2. **Vacuum indexes.** For each index, scan it and delete entries pointing at the collected IDs.
3. **Vacuum heap.** Return to the pages and mark those line pointers **unused**, so they can be
   recycled. Update the free space map and the visibility map.
4. **Truncate** empty pages at the end of the file, if it can briefly get a lock.
5. Update `relfrozenxid`, `pg_class.reltuples` and statistics.

If the dead-ID memory fills up, phases 2 and 3 run, then scanning resumes — which is why "index
scans: N" larger than 1 means VACUUM had to read every index multiple times, a hint to raise
`maintenance_work_mem`.

Pruning (the per-page part of step 1) also happens opportunistically during normal queries, when a
backend reads a page that has enough garbage and enough reason to clean it; that is part of how
HOT updates stay cheap (lesson 3).

## Common mistakes

- Turning autovacuum off "for performance", then discovering bloat or a wraparound shutdown.
- Running `VACUUM FULL` on a live production table and blocking all traffic.
- Ignoring "dead but not yet removable" — the cleanup is being blocked by something old.
- Leaving sessions idle in transaction; not setting `idle_in_transaction_session_timeout`.
- Leaving an unused replication slot in place.
- Expecting `DELETE` or plain VACUUM to make the file smaller.
- Using one global autovacuum setting for a cluster with a few huge, very hot tables.

## Exercise

1. Recreate the `vac` experiment. After the first full-table update, run `VACUUM` and record the
   size; do three more full updates with VACUUM in between and show the size stays flat.
2. Delete the last 90% of rows (highest IDs), vacuum, and check whether the file shrank. Then delete
   rows scattered across the table instead and compare. Explain the difference.
3. Reproduce the "dead but not yet removable" output using an idle Repeatable Read session, then set
   `idle_in_transaction_session_timeout = '10s'` for that session's role and watch it get terminated.
4. Write a monitoring query that returns any table whose `n_dead_tup` exceeds 20% of `n_live_tup`
   and whose last autovacuum was more than an hour ago.
5. Find the five tables with the oldest `relfrozenxid` in your database and run `VACUUM (FREEZE,
   VERBOSE)` on one. What changed in its output and in `age(relfrozenxid)`?
