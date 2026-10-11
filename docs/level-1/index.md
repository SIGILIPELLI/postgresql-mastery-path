---
description: "PostgreSQL Level 1: server architecture, psql, data types, schemas, roles and pg_hba.conf, constraints, isolation levels, upserts, MERGE and pg_dump."
---

# Level 1 · PostgreSQL Foundations

You can already write SQL. Level 1 is about writing it *for PostgreSQL* — using the database's
own tools, types and guarantees instead of treating it as a generic SQL engine. By the end you will
run your own server, find your way around it from `psql`, design tables whose types and constraints
reject bad data on their own, understand what concurrent transactions can and cannot see, and take
backups you have actually restored.

The theme running through every lesson: **push rules into the database**. A constraint is enforced
for every client, every script and every bug; the same rule in application code is enforced only
when that code runs, and often has a race condition. The project at the end proves it with a
ticketing system that survives 40 simultaneous buyers fighting for 5 seats.

**Prerequisites:** everyday SQL — `SELECT`, joins, `GROUP BY`, `CREATE TABLE`. The
[SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/) covers those. Some lessons use a
few lines of Python to run concurrent sessions; the
[Python Mastery Path](https://sigilipelli.github.io/python-mastery-path/) is enough background.

**Setup:** every command was run on PostgreSQL 18.6. Lesson 1 shows how to install it and create a
throwaway cluster; versions 16 and 17 work for nearly everything, and features new in 18 are marked.

## Modules

1. **[How PostgreSQL Is Built: Clusters, Processes & Setup](01-architecture-and-setup.md)** — the
   postmaster, one process per connection, the data directory, and a cluster you create by hand.
2. **[psql Like a Pro](02-psql-like-a-pro.md)** — `\d` commands, `\gset`, `\gexec`, `\copy`,
   `\watch`, and scripts that stop on errors.
3. **[PostgreSQL Data Types That Matter](03-data-types.md)** — numeric vs float, text, `timestamptz`,
   identity columns, `uuidv7()`, enums, arrays and ranges.
4. **[Schemas, search_path & Organizing Objects](04-schemas-search-path.md)** — namespaces, name
   resolution and why the path is a security setting.
5. **[Roles, Privileges & Client Authentication](05-roles-privileges.md)** — group roles, default
   privileges, predefined roles, SCRAM and `pg_hba.conf`.
6. **[Constraints Beyond the Basics](06-constraints.md)** — exclusion constraints, deferrable keys,
   `NOT VALID`, partial uniqueness and generated columns.
7. **[Transactions & Isolation Levels in Practice](07-transactions-isolation.md)** — lost updates,
   write skew, serialization failures and savepoints, reproduced live.
8. **[PostgreSQL-Flavoured SQL](08-postgres-sql-features.md)** — `RETURNING`, `ON CONFLICT`, `MERGE`,
   `DISTINCT ON`, `FILTER`, `generate_series` and `COPY`.
9. **[Backups with pg_dump & pg_restore](09-pg-dump-restore.md)** — formats, parallelism, roles, and
   restoring a single table without losing its indexes.
10. **[Project — Event Ticketing Database](10-project-event-ticketing.md)** — a schema where the
    database prevents double bookings and double sales, stress-tested and backed up.

## After Level 1

Level 2 opens the hood: how MVCC stores row versions, why VACUUM exists, how indexes are laid out,
and how to read `EXPLAIN ANALYZE` well enough to fix slow queries.
