---
description: "Major upgrades with pg_upgrade (a real 17 to 18 run with its checksum gotcha), logical upgrades, and how automated failover works."
---

# 09 · Upgrades & High Availability

Two operational topics that are closely linked: **major-version upgrades**, which every PostgreSQL
installation must do roughly every five years before its version reaches end of life, and **high
availability**, which decides how much downtime any planned or unplanned event — including an upgrade
— costs you.

This lesson performs a real `pg_upgrade` from PostgreSQL 17.11 to 18.6 (both from Homebrew) and then
explains the tooling for automated failover, which was not run here.

## Versions and support

- **Minor releases** (18.5 → 18.6) are bug and security fixes. Same on-disk format: install the new
  binaries and restart. Do them promptly (lesson 8).
- **Major releases** (17 → 18) arrive yearly and may change the on-disk format of system catalogs.
  Each major version is supported for five years. Upgrading needs one of three methods.

| Method | Downtime | Notes |
|--------|----------|-------|
| `pg_dump` / `pg_restore` | proportional to data size (hours for large DBs) | simplest, cleans up bloat, works across platforms |
| `pg_upgrade` | minutes, mostly independent of data size with `--link` | rewrites catalogs, reuses data files |
| logical replication | seconds (a switchover) | most moving parts; sequences, DDL and large objects need care (lesson 4) |

## pg_upgrade, for real

The old cluster: PostgreSQL 17.11 with a 586 MB data directory (a `pgbench` database with 2 million
accounts and a `pg_trgm` index). Stop it, then create the new cluster with the new binaries and run the
compatibility check:

```bash
initdb -D new18 -U postgres -E UTF8 --locale=en_US.UTF-8      # PostgreSQL 18 initdb
pg_upgrade --check -b /opt/homebrew/opt/postgresql@17/bin -B /opt/homebrew/opt/postgresql@18/bin \
           -d old17 -D new18 -U postgres -p 54417 -P 54418
```

```text
Performing Consistency Checks
-----------------------------
Checking cluster versions                                     ok

old cluster does not use data checksums but the new one does
Failure, exiting
```

The first real-world gotcha: **PostgreSQL 18's `initdb` enables data checksums by default** (Level 2 ·
01), PostgreSQL 17's did not, and `pg_upgrade` requires both clusters to match. Options: create the new
cluster with `--no-data-checksums` (as below), or enable checksums on the old cluster first with the
offline `pg_checksums --enable` tool, which rewrites every block and takes time proportional to the data
size.

A second, environment-specific failure: `pg_upgrade` starts both servers with a Unix socket in the
current directory, and my working directory path was too long (`Unix-domain socket path ... is too long
(maximum 103 bytes)`). The `-s` option chooses a short socket directory.

With `--no-data-checksums` and `-s /tmp/pgu`:

```text
Performing Consistency Checks
-----------------------------
Checking cluster versions                                     ok
Checking database connection settings                         ok
Checking database user is the install user                    ok
Checking for prepared transactions                            ok
Checking for contrib/isn with bigint-passing mismatch         ok
Checking for valid logical replication slots                  ok
Checking for subscription state                               ok
Checking data type usage                                      ok
Checking for objects affected by Unicode update               ok
Checking for not-null constraint inconsistencies              ok
Checking for presence of required libraries                   ok
...
*Clusters are compatible*
```

Then the upgrade itself, in the default **copy** mode:

```text
Copying old pg_xact to new server                             ok
Restoring global objects in the new cluster                   ok
Restoring database schemas in the new cluster                 ok
Copying user relation files                                   ok
Upgrade Complete
Some statistics are not transferred by pg_upgrade.
    vacuumdb -U postgres --all --analyze-in-stages --missing-stats-only
    vacuumdb -U postgres --all --analyze-only
    ./delete_old_cluster.sh
real 2.69
```

Verification on the new server:

```text
PostgreSQL 18.6 (Homebrew) on aarch64-apple-darwin27.0.0, ...
2000000                       -- rows in pgbench_accounts
4                             -- pg_stats rows for pgbench_accounts
pg_trgm|1.6
```

The data is intact, and — new in PostgreSQL 18 — **planner statistics were carried over** (4 column
statistics rows exist without running `ANALYZE`), so queries plan sensibly from the first minute. Before
18, a freshly upgraded database had no statistics at all until `ANALYZE` finished, which on large
databases meant a period of terrible plans; `--analyze-in-stages` exists to get rough statistics quickly.
Extended statistics are not transferred, which is why the output still suggests the `vacuumdb` runs.

The `pg_trgm` extension's version did not change (1.6 is also 18's default here); when a new release ships
a newer extension version, run `ALTER EXTENSION ... UPDATE` in each database afterwards.

### --link mode

`--link` hard-links the old data files into the new cluster instead of copying them:

```text
Linking user relation files                                   ok
Upgrade Complete
real 2.40
```

For this 586 MB cluster the difference was small (2.40 s versus 2.69 s), because catalog work dominates
small databases. The point of `--link` is that its time does not grow with data size: copy mode on a
multi-terabyte cluster takes hours and needs twice the disk, link mode typically minutes. The catch, as
`pg_upgrade` warns: once the new cluster has started, **the old one cannot be used** — they share files.
Your fallback is then a backup or a standby that was not upgraded. (PostgreSQL 18 also added a `--swap`
mode that moves directories instead of linking files.)

### A pg_upgrade runbook

1. Read the release notes for every major version you are crossing — look for incompatibilities.
2. Upgrade extensions to versions available for the new major version; install the new PostgreSQL
   packages alongside the old.
3. Rehearse on a copy of production. Run `pg_upgrade --check` against the real cluster (it is
   read-only). Time the rehearsal.
4. Make sure you have a fresh backup (and WAL archive) of the old cluster.
5. Stop the application, stop the old cluster, run `pg_upgrade --link`.
6. Start the new cluster, run the suggested `vacuumdb` commands, smoke-test, open to traffic.
7. Rebuild standbys: physical replicas must run the same major version. (`pg_upgrade` can also upgrade
   standbys with `rsync` in a documented way, which is easy to get wrong; re-cloning is simpler.)

## Upgrading with logical replication

For near-zero downtime: build a new-version server, create the schema there, subscribe it to a
publication of all tables on the old one (lesson 4), let it catch up, then **switch over**: stop writes on
the old server, wait for the subscriber to have applied everything, copy sequence values
(`setval` from `pg_sequences`), and point the application at the new server. Rehearse it — the failure
modes are the ones lesson 4 showed: DDL during the migration, missing replica identities, and sequences.
PostgreSQL 17's `pg_createsubscriber` can turn a physical standby into a logical subscriber, avoiding
the initial data copy for large databases.

## High availability: from standby to automatic failover

Lesson 3 built a standby and promoted it by hand. Doing that automatically, safely, needs three
capabilities that PostgreSQL itself does not provide:

1. **Failure detection** that is not fooled by a network partition — the old primary may still be alive
   and accepting writes from clients that can reach it.
2. **Leader election** with a single source of truth, so exactly one node is primary.
3. **Fencing and reconfiguration** — demote or isolate the old primary, repoint standbys and clients.

The most widely used open-source tool is **Patroni**, which runs alongside each PostgreSQL node and uses a
distributed consensus store (etcd, Consul or ZooKeeper) to hold the leader lock; a node that loses
contact with the store demotes itself. Others include repmgr and pg_auto_failover, and Kubernetes
operators (CloudNativePG, Crunchy PGO and others) that embed similar logic. Managed services provide it
as a feature. None of these were run for this course; evaluate them against your own failure scenarios.

Whatever the tool, the decisions are yours:

- **RPO** (how much data you may lose): asynchronous replication can lose the last moments of commits on
  failover; synchronous replication with at least two candidates (lesson 3) can lose none, at a latency
  cost.
- **RTO** (how long recovery may take): detection timeouts plus promotion plus client reconnection —
  typically tens of seconds with automation.
- **Client routing**: a virtual IP, DNS, a proxy (HAProxy, PgBouncer), or libpq multi-host connection
  strings with `target_session_attrs=read-write`, which try hosts in turn until one accepts writes.
- **Rejoining the old primary**: `pg_rewind` (needs checksums or `wal_log_hints`) or a fresh clone.

And test failover regularly, in production-like conditions — an HA setup that has never failed over is
an untested hypothesis.

## How It Actually Works

`pg_upgrade` does not convert user data. Table and index file formats rarely change between major
versions; the system catalogs do. So it dumps the old cluster's **schema only** (`pg_dump --binary-upgrade`),
restores it into the new cluster in a special mode that preserves every object's OID and relfilenode,
copies the commit log (`pg_xact`) and multixact data so that existing tuples' transaction IDs keep their
meaning, sets the new cluster's next transaction ID past the old one's, and then copies, links, clones or
swaps the data files into place. Because the files are reused byte for byte, anything that changes their
interpretation — a different checksum setting, a different block size, a collation library change that
alters text sort order — must be detected by `--check` or handled by you (the "Unicode update" and
collation checks exist for exactly that).

Automatic failover systems build on the same primitives as lesson 3 — `pg_promote()`, `primary_conninfo`,
`pg_rewind` — wrapped in a control loop driven by a consensus store that guarantees at most one leader.

## Common mistakes

- Running an end-of-life major version because "upgrades are scary".
- Skipping `pg_upgrade --check` and the rehearsal.
- Using `--link` without a fallback backup, then needing to roll back.
- Forgetting to analyze (before 18) or to update extensions after the upgrade.
- Hand-rolled failover scripts without fencing — split brain.
- Never testing failover.

## Exercise

1. Install two major versions side by side, create a cluster with the older one, load data and an
   extension, and upgrade it with `pg_upgrade --check`, then `--link`. Record every warning and how you
   resolved it.
2. Perform the same upgrade with `pg_dump`/`pg_restore -j` and compare total downtime.
3. Write the switchover checklist for a logical-replication upgrade, including how you will verify that the
   subscriber has applied every change before you move traffic.
4. Design (on paper) an HA setup for a database with an RPO of zero and an RTO of one minute: number of
   nodes and zones, synchronous settings, consensus store, client routing, and the failover test you would
   run every quarter.
