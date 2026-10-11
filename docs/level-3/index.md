---
description: "PostgreSQL Level 3: JSONB, full-text search, PL/pgSQL, triggers, LATERAL and recursive queries, partitioning, RLS, extensions and queues."
---

# Level 3 · Advanced Features

PostgreSQL is often described as "more than a database", and this level is where that becomes
concrete. Documents in JSONB, a search engine in `tsvector`, server-side logic in PL/pgSQL, audit
trails from triggers, partitioned tables that drop a month of data in milliseconds, tenant isolation
enforced by the database itself, vector similarity search, and a job queue — each one a feature that
teams frequently bolt on as a separate system.

Every lesson shows the feature working *and* where it bites: operator precedence in JSONB updates,
settings that silently turn into empty strings, views that leak every tenant's rows, default
partitions that block maintenance, an RLS rule that quietly stops a GIN index being used. Those came
from actually building the examples on PostgreSQL 18.6, and they are the details that matter in
production. The project combines most of the level into a multi-tenant SaaS backend, tested as the
application role would use it.

**Prerequisites:** Levels 1 and 2 — especially roles (L1 · 05), constraints (L1 · 06), indexes
(L2 · 04–05) and EXPLAIN (L2 · 06). Some examples use Python (psycopg) to drive concurrent sessions.

## Modules

1. **[JSONB in Depth](01-jsonb.md)** — operators, jsonpath, `JSON_TABLE`, update costs, and choosing
   between GIN operator classes and expression indexes.
2. **[Full-Text Search](02-full-text-search.md)** — stemming, websearch syntax, weighted ranking,
   snippets, accent-insensitive configurations and GIN.
3. **[PL/pgSQL Functions & Procedures](03-plpgsql.md)** — volatility, SQLSTATE errors, set-based
   thinking, procedures that commit, and injection-proof dynamic SQL.
4. **[Triggers & Audit Logging](04-triggers-audit.md)** — `updated_at`, a JSONB diff audit log,
   transition tables, trigger overhead and event triggers.
5. **[Advanced Querying](05-advanced-querying.md)** — LATERAL top-N, recursive CTEs with `SEARCH` and
   `CYCLE`, `ROLLUP`, time-based window frames and writable CTEs.
6. **[Declarative Partitioning](06-partitioning.md)** — pruning, unique-key rules, retention by
   dropping partitions, and the default-partition trap.
7. **[Row-Level Security for Multi-Tenant Apps](07-row-level-security.md)** — policies, owner and view
   bypasses, and connection-pool leaks.
8. **[Extensions Worth Knowing](08-extensions.md)** — pgcrypto, citext, postgres_fdw, pg_buffercache,
   amcheck and pgvector.
9. **[LISTEN/NOTIFY & Job Queues in Postgres](09-listen-notify-queues.md)** — a crash-safe queue with
   `SKIP LOCKED`, retries and back-off.
10. **[Project — Multi-Tenant SaaS Backend](10-project-multi-tenant-saas.md)** — RLS, search, labels,
    auditing and a partitioned history table, tested end to end.

## After Level 3

Level 4 moves from the schema to the server: configuration and memory, connection pooling,
replication, point-in-time recovery, monitoring, zero-downtime migrations, security hardening and
upgrades — ending with a production-readiness capstone.
