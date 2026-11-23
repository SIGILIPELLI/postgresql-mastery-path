---
title: "PostgreSQL Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free PostgreSQL cheat sheet: 23 quick-reference tables from every level of the PostgreSQL Mastery Path, each linked to its full lesson."
---

# PostgreSQL Cheat Sheet

Quick-reference tables from **23 lessons** of the free PostgreSQL Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · PostgreSQL Foundations

### [01 · How PostgreSQL Is Built: Clusters, Processes &amp; Setup](level-1/01-architecture-and-setup.md)

*Vocabulary: cluster, database, schema*

| Term | What it is | Created by |
|------|------------|------------|
| **Cluster** | One running server: one data directory, one port, one set of roles, many databases | `initdb` |
| **Database** | An isolated namespace inside the cluster. A connection belongs to exactly one database | `CREATE DATABASE` |
| **Schema** | A folder for tables, functions and types *inside* one database | `CREATE SCHEMA` |

### [02 · psql Like a Pro](level-1/02-psql-like-a-pro.md)

*Describing things: the `\d` family*

| Command | Lists |
|---------|-------|
| `\l` | databases |
| `\dn` | schemas |
| `\dt`, `\dv`, `\dm`, `\ds` | tables, views, materialized views, sequences |
| `\di` | indexes |
| `\df` | functions |
| `\du` | roles |
| `\dx` | installed extensions |
| `\dp` | access privileges |
| `\d+ name` | an object with extra detail: storage, size, description |

### [03 · PostgreSQL Data Types That Matter](level-1/03-data-types.md)

*Numbers: exact or approximate?*

| Use case | Type |
|----------|------|
| Money, quantities with fixed decimals, anything audited | `numeric(12,2)` or integer cents in `bigint` |
| Measurements, scientific data, ML features | `double precision` |
| Counters, IDs | `integer` or `bigint` |

### [04 · Schemas, search_path &amp; Organizing Objects](level-1/04-schemas-search-path.md)

*A practical layout*

| Schema | Contains | Who can create |
|--------|----------|----------------|
| `app` | application tables, owned by a migration role | migration role only |
| `api` | views/functions exposed to the app or a REST layer | migration role |
| `reporting` | materialized views for dashboards | analytics role |
| `extensions` | extension objects (`CREATE EXTENSION pg_trgm SCHEMA extensions`) | superuser/owner |
| `audit` | audit-log tables written by triggers | nobody but triggers |

### [05 · Roles, Privileges &amp; Client Authentication](level-1/05-roles-privileges.md)

*Predefined roles*

| Role | Grants |
|------|--------|
| `pg_read_all_data` / `pg_write_all_data` | read/write every table in every schema (14+) |
| `pg_monitor` | read monitoring views such as full `pg_stat_activity` |
| `pg_signal_backend` | cancel or terminate other sessions (not superusers') |
| `pg_read_server_files`, `pg_write_server_files`, `pg_execute_server_program` | server-side `COPY` to files/programs — effectively superuser-equivalent, grant with great care |
| `pg_maintain` | VACUUM, ANALYZE, REINDEX etc. on all relations (17+) |

### [07 · Transactions &amp; Isolation Levels in Practice](level-1/07-transactions-isolation.md)

*The three levels PostgreSQL actually has*

| Level | Snapshot taken | Prevents | Still allows |
|-------|----------------|----------|--------------|
| Read Committed (default) | at the start of **each statement** | dirty reads | non-repeatable reads, lost updates in read-modify-write code, write skew |
| Repeatable Read | at the **first statement** of the transaction | the above + non-repeatable and phantom reads, lost updates (by raising an error) | write skew |
| Serializable | like Repeatable Read, plus dependency tracking | every anomaly: the result equals *some* serial order | — (but transactions can fail and must be retried) |

### [09 · Backups with pg_dump &amp; pg_restore](level-1/09-pg-dump-restore.md)

*The four formats*

| Format | Restore with | Compressed | Parallel dump | Parallel restore | Selective restore |
|--------|--------------|-----------|---------------|------------------|-------------------|
| plain (`-Fp`) | `psql` | no (pipe through gzip) | no | no | by editing text |
| custom (`-Fc`) | `pg_restore` | yes | no | yes | yes |
| directory (`-Fd`) | `pg_restore` | yes | **yes** | yes | yes |
| tar (`-Ft`) | `pg_restore` | no | no | no | yes |

## Level 2 · Internals: MVCC, Storage & Indexes

### [02 · VACUUM, Autovacuum &amp; Bloat](level-2/02-vacuum-autovacuum.md)

*Freezing and wraparound*

| Age | What happens |
|-----|--------------|
| `autovacuum_freeze_max_age` (200 million) | autovacuum forces an **aggressive** anti-wraparound vacuum of the table, even if autovacuum is disabled for it |
| `vacuum_failsafe_age` (1.6 billion) | VACUUM drops its cost throttling and skips index cleanup to finish freezing as fast as possible |
| about 2 billion minus 3 million | the server **refuses to assign new transaction IDs** — no writes — until a VACUUM completes |

### [05 · GIN, GiST, BRIN &amp; Hash Indexes](level-2/05-other-index-types.md)

*Choosing*

| Question | Index |
|----------|-------|
| equality, ranges, sorting, uniqueness | B-tree |
| array/JSONB contains, key exists, full-text match | GIN |
| `LIKE '%x%'`, similarity, regex | GIN (or GiST) with `pg_trgm` |
| ranges overlap, exclusion constraints, geometry, nearest neighbour | GiST |
| huge append-only table, filter on time or ID | BRIN |
| equality only on long values | hash (or B-tree) |

### [06 · Reading EXPLAIN ANALYZE](level-2/06-explain-analyze.md)

*Join methods*

| Node | How it works | Good when |
|------|--------------|-----------|
| **Nested Loop** | for each outer row, look up matches in the inner side (ideally via an index) | outer side is small; inner side has an index on the join key |
| **Hash Join** | build a hash table from the smaller input, then probe it with each row of the other | large unsorted inputs, equality joins |
| **Merge Join** | walk two inputs sorted on the join key in step | both inputs already sorted (e.g. from indexes), or very large inputs |

### [08 · Locks, Deadlocks &amp; SKIP LOCKED](level-2/08-locking.md)

*Two families of locks*

| Lock mode | Taken by | Conflicts with |
|-----------|----------|----------------|
| ACCESS SHARE | `SELECT` | ACCESS EXCLUSIVE only |
| ROW SHARE | `SELECT ... FOR UPDATE/SHARE` | EXCLUSIVE, ACCESS EXCLUSIVE |
| ROW EXCLUSIVE | `INSERT`, `UPDATE`, `DELETE`, `MERGE` | SHARE and stronger |
| SHARE UPDATE EXCLUSIVE | `VACUUM`, `ANALYZE`, `CREATE INDEX CONCURRENTLY`, some `ALTER TABLE` forms | itself and stronger |
| SHARE | `CREATE INDEX` (non-concurrent) | ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE and the exclusive modes (not other SHARE locks) — **blocks writes** |
| ACCESS EXCLUSIVE | most `ALTER TABLE`, `DROP`, `TRUNCATE`, `VACUUM FULL`, `REFRESH MATERIALIZED VIEW` | **everything, including SELECT** |

### [10 · Project — Diagnose a Slow Database](level-2/10-project-slow-database.md)

*Step 7 — measure again, the same way*

| | Before | After |
|---|---|---|
| Throughput | 29 tps | 22,584 tps |
| Ticket page | 646.8 ms | 0.375 ms |
| Inbox | 166.2 ms | 0.374 ms |
| Dashboard | 225.1 ms | 0.236 ms |
| Login | 64.4 ms | 0.134 ms |
| Reply (write) | 4.4 ms | 0.78 ms |

## Level 3 · Advanced Features

### [01 · JSONB in Depth](level-3/01-jsonb.md)

*Reading values*

| Operator | Returns |
|----------|---------|
| `->  'key'` / `-> 2` | `jsonb` value of a key / array element |
| `->> 'key'` | that value as `text` |
| `#> '{a,b,0}'` / `#>> '{a,b,0}'` | value at a path, as `jsonb` / `text` |
| `@>` | contains (left contains right) |
| `?`, `?|`, `?&` | key exists / any of / all of |
| `@?`, `@@` | jsonpath exists / jsonpath predicate |

### [03 · PL/pgSQL Functions &amp; Procedures](level-3/03-plpgsql.md)

*Volatility is a promise to the planner*

| Category | Promise | Examples |
|----------|---------|----------|
| `IMMUTABLE` | same arguments → same result, forever; no database access | `lower()`, `vat()` above |
| `STABLE` | same result within one statement; may read the database | `now()`, lookups |
| `VOLATILE` | anything goes; may change between calls or have side effects | `random()`, anything that writes |

### [04 · Triggers &amp; Audit Logging](level-3/04-triggers-audit.md)

*The four kinds of trigger*

| | `FOR EACH ROW` | `FOR EACH STATEMENT` |
|---|---|---|
| **BEFORE** | can modify `NEW` or skip the row (`RETURN NULL`) | runs once before the statement |
| **AFTER** | sees the final row; return value ignored | runs once after; can read **transition tables** |

### [08 · Extensions Worth Knowing](level-3/08-extensions.md)

*Other notable extensions (not run here)*

| Extension | What it does |
|-----------|--------------|
| PostGIS | geographic types, spatial indexes and thousands of GIS functions — the reference spatial database |
| TimescaleDB | time-series hypertables, compression, continuous aggregates (licensing differs by feature) |
| pg_partman | automatic creation and retention of partitions (lesson 6) |
| pg_cron | cron-style job scheduling inside the database |
| pg_repack | rebuild bloated tables and indexes with only brief locks |
| pgaudit | detailed session and object audit logging for compliance |

### [10 · Project — Multi-Tenant SaaS Backend](level-3/10-project-multi-tenant-saas.md)

| Need | Feature |
|------|---------|
| tenant isolation that survives application bugs | row-level security (lesson 7) |
| search box | stored, weighted `tsvector` + GIN (lesson 2) |
| label filters | `text[]` + GIN (Level 2 · 05) |
| `updated_at` and audit trail | triggers (lesson 4) |
| who changed it | per-transaction settings (lessons 4 and 7) |
| cheap retention of history | range partitioning (lesson 6) |
| search API | SQL function (lesson 3) |

## Level 4 · Operations & Production

### [01 · Configuration &amp; Memory Tuning](level-4/01-configuration-tuning.md)

*Where settings come from*

| Context | Change takes effect |
|---------|---------------------|
| `postmaster` | only after a **restart** |
| `sighup` | on reload (`pg_reload_conf()`), for all sessions |
| `superuser` / `user` | reload, or per session with `SET` (superuser-only for the former) |

### [02 · Connections &amp; Pooling](level-4/02-connection-pooling.md)

*Pool modes and their catch*

| Mode | A server connection is assigned to a client... | Compatible with |
|------|----------------------------------------------|-----------------|
| `session` | for the whole client session | everything; little multiplexing |
| `transaction` | for one transaction, then returned | most apps, *if* they avoid session state |
| `statement` | for one statement (multi-statement transactions forbidden) | rarely used |

### [06 · Monitoring a Running Database](level-4/06-monitoring.md)

*What to alert on*

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

### [07 · Zero-Downtime Schema Migrations](level-4/07-zero-downtime-migrations.md)

*Instant, rewrite, or scan?*

| Operation | Effect |
|-----------|--------|
| `ADD COLUMN` (nullable, or constant default) | instant |
| `ADD COLUMN ... DEFAULT volatile()` | rewrite |
| `DROP COLUMN` | instant (space reclaimed later) |
| `ALTER COLUMN TYPE` (most changes) | rewrite + rebuild indexes on it |
| `ALTER COLUMN TYPE varchar(n)` → `text`, or larger `varchar(n)` | instant (binary compatible) |
| `SET NOT NULL` | full scan under ACCESS EXCLUSIVE — unless a validated `CHECK (col IS NOT NULL)` exists |
| `ADD CONSTRAINT ... CHECK / FOREIGN KEY` | full scan under lock — use `NOT VALID` + `VALIDATE` |
| `ADD PRIMARY KEY / UNIQUE` | index build under lock — build the index `CONCURRENTLY` first, then `ADD CONSTRAINT ... USING INDEX` |
| `CREATE INDEX` | blocks writes — use `CONCURRENTLY` |
| `RENAME` table/column | instant, but breaks code that uses the old name |

### [09 · Upgrades &amp; High Availability](level-4/09-upgrades-high-availability.md)

*Versions and support*

| Method | Downtime | Notes |
|--------|----------|-------|
| `pg_dump` / `pg_restore` | proportional to data size (hours for large DBs) | simplest, cleans up bloat, works across platforms |
| `pg_upgrade` | minutes, mostly independent of data size with `--link` | rewrites catalogs, reuses data files |
| logical replication | seconds (a switchover) | most moving parts; sequences, DDL and large objects need care (lesson 4) |

### [10 · Capstone — Production-Ready PostgreSQL](level-4/10-capstone-production-ready.md)

*What "production-ready" means here*

| Requirement | Implemented by | Lesson |
|-------------|----------------|--------|
| Only encrypted, password-verified connections | `hostssl` + `scram-sha-256` only; clients use `verify-full` | 4 · 08 |
| Least privilege | owner / read-write / read-only / monitor / replication roles; no PUBLIC connect | 1 · 05, 4 · 08 |
| Guard rails | per-role `statement_timeout`, `lock_timeout`, read-only role, idle-transaction timeout | 4 · 01 |
| Observability | `pg_stat_statements`, slow-query and DDL logging, a health-check script | 4 · 06 |
| Recoverability | WAL archiving + verified base backups, a timed PITR drill | 4 · 05 |
| Availability | streaming replica over TLS with a capped slot, a failover drill | 4 · 03 |

---

Go deeper with the full [PostgreSQL Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
