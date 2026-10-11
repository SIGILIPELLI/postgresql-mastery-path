---
description: "PostgreSQL Level 4: configuration, pooling, streaming and logical replication, PITR, monitoring, safe migrations, security and upgrades."
---

# Level 4 · Operations & Production

The first three levels were about what happens inside a database. Level 4 is about keeping one alive:
configured sensibly, reachable by thousands of clients, replicated, backed up so that any second can
be recovered, watched, changed without outages, locked down, and upgraded before its version expires.

Every lesson builds the real thing on PostgreSQL 18.6 — a standby that is promoted, a `DROP TABLE` that
is undone by point-in-time recovery, a PgBouncer pool absorbing 200 clients the server refused, a
`pg_upgrade` from 17 to 18 — and records what broke along the way: a synchronous commit that hung when
its standby died, a publication that stopped the application's own `UPDATE`s, an index left behind on
the wrong column after a "zero-downtime" swap, a backup script that carried on after failing. The
capstone puts it all together into a hardened primary-and-replica deployment and proves it with a
recovery drill and a failover drill.

**Prerequisites:** Levels 1–3, especially WAL (Level 2 · 09), locking (Level 2 · 08) and roles
(Level 1 · 05). You need a machine where you can run several PostgreSQL instances on different ports;
some lessons install PgBouncer and a second PostgreSQL major version.

## Modules

1. **[Configuration & Memory Tuning](01-configuration-tuning.md)** — where settings come from, reload
   versus restart, the few settings that matter, and an honest `shared_buffers` benchmark.
2. **[Connections & Pooling](02-connection-pooling.md)** — the cost of connections, PgBouncer pool
   modes, and session state that leaks between clients.
3. **[Streaming Replication](03-streaming-replication.md)** — `pg_basebackup`, slots, lag, synchronous
   commit, and promotion.
4. **[Logical Replication & Change Data Capture](04-logical-replication.md)** — publications with
   filters, replica identity, DDL breakage and decoding changes yourself.
5. **[Point-in-Time Recovery](05-point-in-time-recovery.md)** — WAL archiving, verified base backups and
   recovering to the second before an accident.
6. **[Monitoring a Running Database](06-monitoring.md)** — activity and wait events, progress views,
   `pg_stat_io`, `auto_explain`, and what to alert on.
7. **[Zero-Downtime Schema Migrations](07-zero-downtime-migrations.md)** — measured lock impact,
   `lock_timeout`, `NOT VALID`, and expand/backfill/contract.
8. **[Security Hardening](08-security-hardening.md)** — TLS with verification, `pg_hba.conf`, revoking
   permissive defaults and privilege audits.
9. **[Upgrades & High Availability](09-upgrades-high-availability.md)** — a real `pg_upgrade`, logical
   upgrades, and how automated failover works.
10. **[Capstone — Production-Ready PostgreSQL](10-capstone-production-ready.md)** — build it, then prove
    it with a PITR drill and a failover drill.

## After Level 4

You now have the knowledge to run PostgreSQL in production — and the habit that matters most: verifying
every claim, including your own, by trying it. Related courses in this series include the
[Docker](https://sigilipelli.github.io/docker-mastery-path/) and
[Kubernetes](https://sigilipelli.github.io/kubernetes-mastery-path/) paths for running databases in
containers, and the [Data Engineering Mastery Path](https://sigilipelli.github.io/data-engineering-mastery-path/)
for moving data out of PostgreSQL into analytics systems.
