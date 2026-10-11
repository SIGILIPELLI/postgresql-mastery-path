---
description: "Configure PostgreSQL deliberately: setting sources, ALTER SYSTEM, reload vs restart, per-role settings, memory, and a shared_buffers test."
---

# 01 · Configuration & Memory Tuning

PostgreSQL ships with defaults designed to start on almost anything — a 128 MB buffer cache,
4 MB per sort, settings that assume spinning disks. A production server needs a few of them changed.
Not many: most performance comes from schema, indexes and queries (Levels 2 and 3), and tuning
dozens of knobs by folklore usually achieves nothing measurable. This lesson covers how
configuration works, the handful of settings that matter, and what changing one actually did on a
real test.

Commands ran against PostgreSQL 18.6 on a laptop with 8 GB of RAM and 8 CPU cores.

## Where settings come from

A setting's effective value is decided by layers, later ones winning:

1. compiled-in defaults
2. `postgresql.conf` (and files it `include`s)
3. `postgresql.auto.conf`, written by `ALTER SYSTEM`
4. per-database, per-role, and per-role-in-database settings (`ALTER DATABASE/ROLE ... SET`)
5. client connection options, and `SET` / `SET LOCAL` in the session

`pg_settings` shows each value, its unit, its source, and — crucially — its **context**, which says what
it takes to change it:

```text
SELECT name, setting, unit, context, source FROM pg_settings WHERE name IN (...) ORDER BY context, name;
             name             | setting | unit |  context   |       source
------------------------------+---------+------+------------+--------------------
 huge_pages                   | try     |      | postmaster | default
 io_method                    | worker  |      | postmaster | default
 max_connections              | 100     |      | postmaster | configuration file
 shared_buffers               | 16384   | 8kB  | postmaster | configuration file
 autovacuum_vacuum_cost_limit | -1      |      | sighup     | default
 io_workers                   | 3       |      | sighup     | default
 log_min_duration_statement   | -1      | ms   | superuser  | default
 effective_cache_size         | 524288  | 8kB  | user       | default
 effective_io_concurrency     | 16      |      | user       | default
 maintenance_work_mem         | 65536   | kB   | user       | default
 random_page_cost             | 4       |      | user       | default
 work_mem                     | 4096    | kB   | user       | default
```

| Context | Change takes effect |
|---------|---------------------|
| `postmaster` | only after a **restart** |
| `sighup` | on reload (`pg_reload_conf()`), for all sessions |
| `superuser` / `user` | reload, or per session with `SET` (superuser-only for the former) |

Note the units: `shared_buffers` is stored in 8 kB pages (16384 × 8 kB = 128 MB). Always write values
with units (`'1GB'`, `'64MB'`) to avoid mistakes.

## ALTER SYSTEM, reload and restart

```sql
ALTER SYSTEM SET shared_buffers = '1GB';
ALTER SYSTEM SET work_mem = '16MB';
ALTER SYSTEM SET random_page_cost = 1.1;
SELECT pg_reload_conf();
SELECT name, setting, unit, pending_restart FROM pg_settings
WHERE name IN ('shared_buffers','work_mem','random_page_cost');
```

```text
       name       | setting | unit | pending_restart
------------------+---------+------+-----------------
 random_page_cost | 1.1     |      | f
 shared_buffers   | 16384   | 8kB  | t
 work_mem         | 16384   | kB   | f
```

The reload applied `work_mem` and `random_page_cost` immediately. `shared_buffers` is a postmaster
setting: still 128 MB, flagged `pending_restart`. `pg_file_settings` shows what the files say and
whether each line was applied:

```text
 from_auto_conf |       name       | setting | applied |            error
----------------+------------------+---------+---------+------------------------------
 f              | shared_buffers   | 128MB   | f       |
 t              | shared_buffers   | 1GB     | f       | setting could not be applied
 t              | work_mem         | 16MB    | t       |
 t              | random_page_cost | 1.1     | t       |
```

`ALTER SYSTEM` validates values and names, which hand-editing `postgresql.conf` does not:

```text
ALTER SYSTEM SET work_mem = 'lots';
ERROR:  invalid value for parameter "work_mem": "lots"
ALTER SYSTEM SET shared_bufers = '2GB';
ERROR:  unrecognized configuration parameter "shared_bufers"
```

A typo in `postgresql.conf` itself is only detected at reload or start — and a bad postmaster-level
value can stop the server from starting. Check `pg_file_settings` after editing, before restarting.

Teams usually manage configuration as files in version control (Ansible, Kubernetes operators, or
your provider's parameter groups) rather than with `ALTER SYSTEM`, so the running server matches what
is in Git. Pick one approach and stick to it; settings changed by both are confusing.

### Per-role and per-database settings

```sql
ALTER ROLE app_user SET work_mem = '64MB';
ALTER DATABASE perf SET random_page_cost = 1.5;
SELECT coalesce(d.datname, '*') AS db, coalesce(r.rolname, '*') AS role, s.setconfig
FROM pg_db_role_setting s
LEFT JOIN pg_database d ON d.oid = s.setdatabase
LEFT JOIN pg_roles r ON r.oid = s.setrole ORDER BY 1, 2;
```

```text
    db     |       role       |                    setconfig
-----------+------------------+-------------------------------------------------
 *         | app_user         | {"search_path=app_user, billing",work_mem=64MB}
 perf      | *                | {random_page_cost=1.5}
 ticketing | ticketing_app    | {search_path=tix}
 ticketing | ticketing_report | {search_path=tix}
```

These apply at the start of each new session. They are the right place for "the reporting role gets
more `work_mem`" and for timeouts per role (below).

## The settings that matter

### Memory

| Setting | What it is | Starting point |
|---------|------------|----------------|
| `shared_buffers` | PostgreSQL's own page cache, allocated at start | ~25% of RAM on a dedicated server |
| `effective_cache_size` | the planner's *estimate* of total cache (shared buffers + OS cache); allocates nothing | 50–75% of RAM |
| `work_mem` | memory **per sort/hash node, per process** before spilling to disk | 4–64 MB globally; more for specific roles |
| `maintenance_work_mem` | for `CREATE INDEX`, `VACUUM`, `ALTER TABLE ADD FOREIGN KEY` | 256 MB – 1 GB+ |
| `autovacuum_work_mem` | the same for autovacuum workers (defaults to `maintenance_work_mem`) | |
| `huge_pages` | use huge OS pages for shared memory (Linux) | `try` (default); configure the OS for large `shared_buffers` |

The danger is `work_mem`: a complex query can run several sort and hash nodes at once, in several
parallel workers, in each of hundreds of connections. 64 MB × 4 nodes × 3 processes × 200 connections
is 150 GB — on paper. Keep the global value modest and raise it where you know it is needed.

### Planner

- `random_page_cost = 1.1`–`2` on SSDs (lesson Level 2 · 07); keep 4 for spinning disks.
- `effective_io_concurrency` (default 16 in PostgreSQL 18): how many concurrent prefetch requests to
  issue; higher values suit SSDs and network storage.

### I/O (new in PostgreSQL 18)

PostgreSQL 18 introduced asynchronous I/O. `io_method = worker` (the default, which you saw as the
"io worker" processes in Level 1 · 01) hands reads to background I/O workers; on Linux,
`io_method = io_uring` uses the kernel's io_uring interface instead, and `sync` restores the old
behaviour. Sequential scans, bitmap heap scans and VACUUM benefit first. Measure before changing it,
and expect this area to evolve over the next releases.

### Timeouts — cheap insurance

```sql
ALTER ROLE app_user SET statement_timeout = '30s';
ALTER ROLE app_user SET idle_in_transaction_session_timeout = '60s';
ALTER ROLE app_user SET lock_timeout = '5s';
ALTER ROLE app_user SET transaction_timeout = '5min';  -- PostgreSQL 17+: caps a whole transaction
```

These stop a runaway query, a forgotten open transaction (which blocks VACUUM, Level 2 · 02) and lock
pile-ups (Level 2 · 08) from becoming incidents. Set them per role: the application should not be
allowed what a nightly maintenance job is.

### Logging

```text
log_min_duration_statement = '250ms'   # log slow statements with their duration
log_checkpoints = on                   # default on since 15
log_lock_waits = on                    # log waits longer than deadlock_timeout
log_autovacuum_min_duration = '10s'
log_temp_files = 0                     # log every spill to disk with its size
log_line_prefix = '%m [%p] %q%u@%d '   # time, pid, user@database
```

Lesson 6 builds monitoring on top of these.

## An honest benchmark: does shared_buffers matter?

The most-tuned setting of all. A 756 MB `pgbench` database at scale 50, read-only workload
(`pgbench -S`: random primary-key lookups), 8 clients, 20 seconds per run:

```text
shared_buffers = 128MB   tps = 75027.586814   latency average = 0.107 ms
shared_buffers = 128MB   tps = 73668.497038   latency average = 0.109 ms
                         (restart with shared_buffers = 1GB)
shared_buffers = 1GB     tps = 78397.366907   latency average = 0.102 ms
shared_buffers = 1GB     tps = 83049.593679   latency average = 0.096 ms
```

Around 10% better — with the whole database fitting in 1 GB versus less than a fifth of it in 128 MB.
Why so little? Because PostgreSQL reads through the **operating system's page cache**. With 8 GB of RAM
and a 756 MB database, every page was cached by the OS anyway; a "miss" in `shared_buffers` was a
memory copy, not a disk read. (The first 1 GB run was also partly warming the cache — the second run
was faster.)

The lesson is not "shared_buffers doesn't matter". On a server where the hot data exceeds RAM, or under
heavy write load where checkpoint behaviour and buffer replacement interact, it matters a great deal.
The lesson is: **measure with your workload**, and expect the OS cache to be doing much of the work —
which is why ~25% of RAM, not 90%, is the usual guidance.

## How It Actually Works

Configuration files are parsed by the postmaster at start and on SIGHUP; each backend then re-reads
the effective values when it receives the reload signal at a safe point (between queries). Each
setting has a context that controls when it can change: postmaster-context settings size or configure
things allocated once at start — shared memory, the number of connection slots, the I/O method — so
they cannot change in a running server.

`shared_buffers` is a single shared-memory array of 8 kB buffers plus a descriptor array and a hash
table mapping `(relation, fork, block)` to buffer. A backend needing a page looks it up; on a miss it
picks a victim buffer with the **clock-sweep** algorithm (each buffer has a usage count that is bumped
on access and decremented as the clock hand passes; a buffer at zero is evicted, after being written
out if dirty). Large sequential scans and VACUUM use small **ring buffers** instead of the whole pool,
so one big scan cannot flush everyone's hot data.

Settings like `work_mem` are not allocations; they are limits that each executor node checks as it
accumulates data, switching to an on-disk algorithm when exceeded.

## Common mistakes

- Copying a "tuned postgresql.conf" from a blog without measuring anything.
- `work_mem = 1GB` globally.
- Editing `postgresql.conf` and restarting without checking `pg_file_settings`, then being unable to
  start.
- Changing `shared_buffers` and forgetting it needs a restart (check `pending_restart`).
- Leaving `statement_timeout` and `idle_in_transaction_session_timeout` unset for application roles.
- Settings managed both by `ALTER SYSTEM` and by a config-management tool, overwriting each other.

## Exercise

1. List every setting on your server whose `source` is not `default`. For each, explain why it was
   changed — or reset it.
2. Use `pg_file_settings` to catch a deliberate typo you add to `postgresql.conf` before reloading.
3. Repeat the `pgbench -S` comparison on your own machine at scale 50 and at a scale larger than your
   RAM (if you have the disk), with `shared_buffers` at 128 MB and at 25% of RAM. Explain the
   difference between the two scales.
4. Configure per-role timeouts for an application role and a reporting role, and prove each one with a
   `pg_sleep()` call that exceeds it.
