---
description: "Logical backups done right: pg_dump formats, parallel dumps, pg_dumpall for roles, selective restores and the gotchas that break restores."
---

# 09 · Backups with pg_dump & pg_restore

A backup you have never restored is a hope, not a backup. This lesson covers PostgreSQL's
**logical** backup tools — `pg_dump`, `pg_dumpall` and `pg_restore` — which export the *contents*
of a database as SQL or an archive you can load into any compatible server. They are perfect for
moving databases between machines and versions, for copying production to staging, and for small
to medium backups.

They are not the whole story. A logical dump is a snapshot at one moment; restoring it loses
everything since. Level 4 · 05 adds **physical** backups with continuous WAL archiving, which can
restore to any second. Real systems usually use both.

Commands below were run with PostgreSQL 18.6 client tools against the `shop` database built in
earlier lessons.

## What pg_dump guarantees

`pg_dump` connects as an ordinary client and runs in a single **Repeatable Read** transaction
(lesson 7). Every table is read from the same snapshot, so the dump is consistent — no order without
its customer — even while the application keeps writing. It takes only `ACCESS SHARE` locks, which
block nothing except DDL like `DROP TABLE` or `ALTER TABLE` on the tables it is dumping.

It dumps **one database**. It does not include roles or tablespaces, which are cluster-wide.

## The four formats

```bash
pg_dump -d shop -Fp -f shop.sql          # plain SQL script
pg_dump -d shop -Fc -f shop.dump         # custom: compressed archive
pg_dump -d shop -Fd -j 2 -f shop_dir     # directory: one file per table, parallel
pg_dump -d shop -Ft -f shop.tar          # tar (rarely the best choice)
```

```text
drwx------  13  416  shop_dir
-rw-r--r--   1  17683  shop.dump
-rw-r--r--   1  33046  shop.sql
```

| Format | Restore with | Compressed | Parallel dump | Parallel restore | Selective restore |
|--------|--------------|-----------|---------------|------------------|-------------------|
| plain (`-Fp`) | `psql` | no (pipe through gzip) | no | no | by editing text |
| custom (`-Fc`) | `pg_restore` | yes | no | yes | yes |
| directory (`-Fd`) | `pg_restore` | yes | **yes** | yes | yes |
| tar (`-Ft`) | `pg_restore` | no | no | no | yes |

Use **custom** as your default, and **directory with `-j`** when the database is large enough that
dump time matters. Use **plain** when a human needs to read or edit the result.

The plain script starts with session settings that make it reproducible regardless of the target's
configuration:

```sql
\restrict akuIp7wrpjstw1jXzC8w7vlxh31w03enEcv051xWVtHkMo96Fb5zwTgSB2ts7pJ
SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET transaction_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
...
CREATE SCHEMA app;
ALTER SCHEMA app OWNER TO migrator;
```

Note the empty `search_path` — every object in the dump is schema-qualified, so a hostile function
in the target's `public` schema cannot hijack the restore (lesson 4). The `\restrict` /
`\unrestrict` lines with a random key are recent: they were added in the 2025 minor releases as a
security fix, and stop a malicious dump from smuggling psql meta-commands into your restore session.
psql versions older than those releases do not understand them; restore modern plain dumps with a
current psql.

Data is loaded with `COPY ... FROM stdin` blocks — fast, as lesson 8 showed:

```text
COPY public.customers (id, email) FROM stdin;
```

## Restoring

Plain format goes through psql. Archive formats go through `pg_restore`:

```bash
createdb shop_restored
pg_restore -d shop_restored -j 2 shop.dump
psql -d shop_restored -Atc "select count(*) from customers"   # 1000
```

Mixing them up gives a clear error:

```text
$ pg_restore -d shop_r2 shop.sql
pg_restore: error: input file appears to be a text format dump. Please use psql.
```

Restore options you will use:

- `-j N` — parallel restore (custom or directory format). Data loading *and* index builds run in
  parallel; often the biggest speed-up available.
- `--no-owner` — do not try to `ALTER ... OWNER TO` the original roles; everything is owned by the
  restoring user. Essential when restoring into an environment that lacks those roles.
- `--no-acl` (`-x`) — skip `GRANT`/`REVOKE`.
- `--clean --if-exists` — drop objects before recreating them (for restoring over an existing copy).
- `--exit-on-error` (`-e`) — stop at the first error instead of reporting a count at the end.
- `-1` / `--single-transaction` — all or nothing (not combinable with `-j`).

By default `pg_restore` keeps going after errors and exits non-zero only at the end, so in scripts
always check the exit code and read the error summary.

## Roles: pg_dumpall --globals-only

Restoring `shop` into a brand-new cluster fails on every `ALTER ... OWNER TO migrator` and
`GRANT ... TO readers`, because roles are not in the dump. Dump them separately:

```bash
pg_dumpall --globals-only -f globals.sql
```

```text
CREATE ROLE alice;
ALTER ROLE alice WITH NOSUPERUSER INHERIT NOCREATEROLE NOCREATEDB LOGIN NOREPLICATION NOBYPASSRLS PASSWORD 'SCRAM-SHA-256$...(trimmed)';
CREATE ROLE migrator;
...
CREATE ROLE readers;
ALTER ROLE readers WITH NOSUPERUSER INHERIT NOCREATEROLE NOCREATEDB NOLOGIN NOREPLICATION NOBYPASSRLS;
```

The file contains password verifiers — treat it like a secret. A complete logical backup of a
cluster is therefore: `globals.sql` + one `pg_dump -Fc` per database. (`pg_dumpall` without
`--globals-only` dumps everything as one giant plain script, which cannot be restored in parallel
or selectively; avoid it for anything large.)

## Selective restores — and a real gotcha

The archive formats have a table of contents you can list:

```text
$ pg_restore -l shop.dump
13; 2615 24643 SCHEMA - app migrator
3930; 0 0 ACL - SCHEMA app migrator
12; 2615 24633 SCHEMA - app_user app_user
10; 2615 24610 SCHEMA - billing postgres
...
241; 1259 24671 TABLE app legacy migrator
```

Someone dropped `customers` in staging and you want just that table back. The obvious command:

```bash
pg_restore -d shop_r2 -t customers shop.dump
psql -d shop_r2 -c "\d customers"
```

```text
             Table "public.customers"
 Column |  Type  | Collation | Nullable | Default
--------+--------+-----------+----------+---------
 id     | bigint |           | not null |
 email  | text   |           | not null |
```

The rows are back (1,000 of them) — but the **primary key is gone, and so is the identity**.
`-t` selects only the table definition and its data; indexes, constraints, the sequence and
triggers are separate archive entries that `-t` does not pull in. Restoring a table this way and
going back to production traffic is how duplicate IDs happen.

The reliable method is to build a list file containing every entry you need, and restore from it:

```bash
pg_restore -l shop.dump | grep -E " customers( |_)" > customers.list
cat customers.list
```

```text
228; 1259 16386 TABLE public customers postgres
227; 1259 16385 SEQUENCE public customers_id_seq postgres
3910; 0 16386 TABLE DATA public customers postgres
3939; 0 0 SEQUENCE SET public customers_id_seq postgres
3749; 2606 16394 CONSTRAINT public customers customers_pkey postgres
```

```bash
pg_restore -d shop_r2 -L customers.list shop.dump
```

```text
                       Table "public.customers"
 Column |  Type  | Collation | Nullable |           Default
--------+--------+-----------+----------+------------------------------
 id     | bigint |           | not null | generated always as identity
 email  | text   |           | not null |
Indexes:
    "customers_pkey" PRIMARY KEY, btree (id)
```

Review the list by hand for indexes, foreign keys, triggers and grants named differently from the
table. The list-file approach also lets you *reorder* or comment out entries (prefix with `;`).

## Other useful pg_dump options

- `-n schema` / `-N schema`, `-t table` / `-T table` — include or exclude (patterns allowed).
- `--exclude-table-data='audit.*'` — keep the definition, skip the (huge) data.
- `-s` (schema only) — for diffing schemas between environments.
- `--no-statistics` / `--statistics` — PostgreSQL 18 can dump planner statistics so a restored
  database does not need a full `ANALYZE` before it performs well. In 18.6's `pg_dump`, a plain dump
  of `shop` contained no statistics unless asked (I found zero `pg_restore_relation_stats` calls), so
  run `ANALYZE` after any restore, or pass `--statistics` and verify.
- `-Z` / `--compress` — choose method and level, e.g. `-Z zstd:5` for custom/directory formats.

## Versions

`pg_dump` can dump servers **older** than itself, and its output can be restored into the same or a
**newer** major version. Always use the newest client tools available — dumping a PostgreSQL 13
server with `pg_dump` 18 and restoring into 18 is the standard way to do a dump-based upgrade
(Level 4 · 09 compares this with `pg_upgrade`). A newer server cannot be dumped by an older
`pg_dump`.

## A minimal backup script

```bash
#!/usr/bin/env bash
set -euo pipefail
stamp=$(date -u +%Y%m%dT%H%M%SZ)
dest=/backups/$stamp
mkdir -p "$dest"

pg_dumpall --globals-only -f "$dest/globals.sql"
for db in $(psql -XAtc "select datname from pg_database where datallowconn and not datistemplate"); do
  pg_dump -d "$db" -Fc -f "$dest/$db.dump"
done

# prove it: restore the main database into a scratch database and run a sanity query
dropdb --if-exists restore_check
createdb restore_check
pg_restore -d restore_check --no-owner -j 4 "$dest/shop.dump"
psql -XAtc "select count(*) from customers" -d restore_check
```

The last four lines are what turn this from "a backup" into "a tested backup". Run the check
automatically and alert when it fails. Then copy the files somewhere that is not this server.

## How It Actually Works

`pg_dump` reads the system catalogs to discover objects, then builds a dependency graph so it can
emit them in a valid order: schemas before tables, tables before indexes, functions before the
views that call them. Each object becomes an entry in the archive's table of contents (`toc.dat` in
directory format) with an ID, a type, a dependency list and its SQL. Data entries hold `COPY`
streams. `pg_restore` walks the TOC; with `-j`, it uses the dependency graph to run independent
entries — two tables' data, or many index builds — at the same time on separate connections.

To make a parallel dump consistent, the leader opens its Repeatable Read transaction, **exports the
snapshot** (`pg_export_snapshot()`), and each worker sets the same snapshot with
`SET TRANSACTION SNAPSHOT`, so all of them read the database as of one instant.

Because a dump holds one snapshot open for its whole duration, a very long dump on a busy server
keeps VACUUM from removing row versions newer than that snapshot (Level 2 · 02). That is another
reason large databases move to physical backups.

## Common mistakes

- Never testing a restore.
- Forgetting `pg_dumpall --globals-only`, then restoring into a cluster without the roles.
- Using `pg_restore -t` for a single-table restore and losing its indexes and constraints.
- Storing backups only on the database server's own disk.
- Restoring a plain dump with an outdated psql, or trying to dump a newer server with an old
  `pg_dump`.
- Ignoring `pg_restore`'s exit status — it continues past errors by default.
- Treating `pg_dump` as the only backup for a database where losing hours of writes is unacceptable.

## Exercise

1. Take custom-format and directory-format (`-j 4`) dumps of your lab database; compare sizes and
   times.
2. Create a new cluster on a different port (lesson 1), restore `globals.sql` and then the dump with
   `-j`. Count rows in three tables on both sides.
3. Drop one table in the restored copy and bring it back using a hand-edited `-L` list file. Verify
   its indexes, constraints, sequence and grants all returned.
4. Run `pg_dump -s` against two databases and `diff` the outputs to find schema drift.
5. Turn the backup script above into a scheduled job (cron or launchd) that also deletes dumps
   older than 14 days and reports failures.
