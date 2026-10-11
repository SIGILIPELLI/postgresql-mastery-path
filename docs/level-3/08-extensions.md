---
description: "Extend PostgreSQL safely: pgcrypto, citext, postgres_fdw, pg_buffercache, amcheck and pgvector with HNSW, plus how extensions install."
---

# 08 · Extensions Worth Knowing

A large part of PostgreSQL's power lives in extensions: packages of types, functions, operators,
index methods and background workers that install into a database with one command. You have
already used several — `pg_trgm`, `btree_gist`, `pageinspect`, `pgstattuple`, `unaccent`,
`pg_stat_statements`. This lesson covers how extensions work and a handful more that solve common
problems, including running pgvector for similarity search.

Everything shown ran on PostgreSQL 18.6 with the contrib modules that ship with it, plus pgvector
0.8.7 installed with Homebrew.

## Installing and inspecting

There are two steps. The extension's files must be **installed on the server's filesystem** (by the
PostgreSQL package for contrib modules, or a separate package for third-party ones), and then
**created in each database** that uses it:

```sql
SELECT count(*) FROM pg_available_extensions;   -- 56 on this build
CREATE EXTENSION pgcrypto;
\dx                                             -- extensions installed in this database
```

An extension's objects belong to it: `DROP EXTENSION` removes them all, and `pg_dump` emits a single
`CREATE EXTENSION` instead of its objects. Its types exist only once it is created in that database —
my first `CREATE TABLE` in this lesson failed with `type "citext" does not exist` because I ran it
before `CREATE EXTENSION citext`, even though the extension's files were installed on the server.

Upgrades happen in two steps too: install the new package files, then `ALTER EXTENSION name UPDATE;`
in each database. `pg_available_extensions.default_version` versus `pg_extension.extversion` shows
which databases lag behind.

**Trusted extensions.** Normally only superusers can create extensions. Extensions marked *trusted*
(such as `pgcrypto`, `citext`, `pg_trgm`, `btree_gist`) can be created by any role with `CREATE`
privilege on the database — which is how the ticketing project's owner role created `btree_gist` in
Level 1 · 10. Managed cloud services usually expose an allow-list of extensions.

Some extensions also need `shared_preload_libraries` and a restart because they hook into the server
at start-up (`pg_stat_statements`, `auto_explain`, `pg_cron`, TimescaleDB).

## pgcrypto and citext: users and passwords

```sql
CREATE EXTENSION pgcrypto;
CREATE EXTENSION citext;
CREATE TABLE app_users (id int PRIMARY KEY, email citext UNIQUE, pw_hash text NOT NULL);

INSERT INTO app_users VALUES (1, 'Ana@Example.com', crypt('correct horse', gen_salt('bf', 10)));

SELECT id, email, left(pw_hash, 7) AS hash_prefix FROM app_users WHERE email = 'ana@example.COM';
```

```text
 id |      email      | hash_prefix
----+-----------------+-------------
  1 | Ana@Example.com | $2a$10$
```

- `citext` compares case-insensitively while preserving the original spelling, so the lookup with
  different case matched, and the unique constraint rejects a case variant:

    ```text
    INSERT INTO app_users VALUES (2, 'ANA@example.com', 'x');
    ERROR:  duplicate key value violates unique constraint "app_users_email_key"
    DETAIL:  Key (email)=(ANA@example.com) already exists.
    ```

    (A unique index on `lower(email)` achieves the same with plain `text`; `citext` saves remembering
    to lower-case every query.)

- `crypt()` with `gen_salt('bf', 10)` produces a bcrypt hash (`$2a$10$` = bcrypt, cost 10). Verify
  by hashing the attempt with the stored hash as salt:

    ```text
    SELECT pw_hash = crypt('correct horse', pw_hash) AS right_pw,
           pw_hash = crypt('wrong', pw_hash)        AS wrong_pw FROM app_users;
     right_pw | wrong_pw
    ----------+----------
     t        | f
    ```

    One caution: the plaintext password travels to the server in the SQL and can appear in logs if
    statement logging is on. Most applications hash in the application instead (with bcrypt, scrypt or
    Argon2 libraries) and store only the hash.

- `digest('hello', 'sha256')` and `hmac()` provide hashes; `pgp_sym_encrypt`/`pgp_sym_decrypt`
  encrypt column values — but the key must come from somewhere, and if it is passed in SQL, it is as
  exposed as the data. Prefer encryption at the application or storage layer for most needs.

## postgres_fdw: query another PostgreSQL database

Foreign data wrappers make remote tables look local. Querying the `shop` database from the `adv`
database on the same server:

```sql
CREATE EXTENSION postgres_fdw;
CREATE SERVER shop_srv FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (host 'localhost', port '54329', dbname 'shop');
CREATE USER MAPPING FOR CURRENT_USER SERVER shop_srv OPTIONS (user 'postgres');
CREATE SCHEMA shop_remote;
IMPORT FOREIGN SCHEMA public LIMIT TO (customers) FROM SERVER shop_srv INTO shop_remote;

SELECT count(*) FROM shop_remote.customers;   -- 1000
EXPLAIN (VERBOSE, COSTS OFF) SELECT email FROM shop_remote.customers WHERE id < 3;
```

```text
 Foreign Scan on shop_remote.customers
   Output: email
   Remote SQL: SELECT email FROM public.customers WHERE ((id < 3))
```

`Remote SQL` shows what was sent: the filter and the column list were **pushed down**, so only matching
rows crossed the connection. Joins between two foreign tables on the same server, aggregates, sorts and
`LIMIT` can be pushed down too. Conditions using functions the remote side might not have (or that are
not marked safe) are not, and then whole tables cross the wire — always check `EXPLAIN VERBOSE`. The
user mapping here relied on this lab's `trust` authentication; a real one carries a password, stored
in the catalog and readable by superusers.

This is how you join across databases (Level 1 · 01 noted plain SQL cannot), run migrations between
servers, or archive old partitions to a cheaper instance.

## pg_buffercache: what is in memory?

```sql
CREATE EXTENSION pg_buffercache;
SELECT c.relname, count(*) AS buffers, pg_size_pretty(count(*) * 8192) AS cached
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
 AND b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
GROUP BY c.relname ORDER BY 2 DESC LIMIT 5;
```

```text
       relname        | buffers | cached
----------------------+---------+---------
 prices               |    3632 | 28 MB
 log                  |    2635 | 21 MB
 kb_big               |    2147 | 17 MB
 prices_pkey          |    1215 | 9720 kB
 measurements_2026_09 |    1202 | 9616 kB
```

It answers "is my hot data actually cached?" and, after a restart, `pg_prewarm` can reload chosen
relations into the cache (with its `autoprewarm` worker it saves and restores the buffer list
automatically).

## amcheck: verify an index is not corrupt

```sql
CREATE EXTENSION amcheck;
SELECT bt_index_check('big_events_pkey', true);   -- returns nothing on success; errors on corruption
```

`bt_index_check` verifies a B-tree's ordering invariants under a light lock; with the second argument
it also checks every heap row has an index entry. `pg_amcheck` is a command-line wrapper for checking
whole databases. Run it after hardware incidents, before and after major upgrades, and when you
suspect a collation change (an OS upgrade that changes glibc or ICU sort order can silently corrupt
text indexes).

## pgvector: similarity search

pgvector adds a `vector` type, distance operators and approximate-nearest-neighbour indexes — the
database side of retrieval-augmented generation and semantic search. Toy 3-dimensional embeddings:

```sql
CREATE EXTENSION vector;                  -- 0.8.7 here
CREATE TABLE items (id int PRIMARY KEY, label text, embedding vector(3));
INSERT INTO items VALUES (1, 'cat', '[0.9, 0.1, 0.0]'), (2, 'kitten', '[0.85, 0.15, 0.05]'),
                         (3, 'car', '[0.1, 0.9, 0.2]'), (4, 'truck', '[0.05, 0.95, 0.3]');

SELECT label, round((embedding <=> '[0.88, 0.12, 0.02]')::numeric, 4) AS cosine_distance
FROM items ORDER BY embedding <=> '[0.88, 0.12, 0.02]' LIMIT 2;
```

```text
 label  | cosine_distance
--------+-----------------
 cat    |          0.0006
 kitten |          0.0014
```

`<=>` is cosine distance, `<->` Euclidean, `<#>` negative inner product. At scale, 50,000 random
64-dimensional vectors:

```sql
CREATE INDEX docs_vec_hnsw ON docs_vec USING hnsw (embedding vector_cosine_ops);   -- 5.7 s
```

```text
-- with the HNSW index (vector literal trimmed)
 Limit (actual time=0.906..0.911 rows=5.00 loops=1)
   ->  Index Scan using docs_vec_hnsw on docs_vec (actual time=0.906..0.910 rows=5.00 loops=1)
         Order By: (embedding <=> '[0.82380086,0.8122004,...]'::vector)
 Execution Time: 0.917 ms

-- exact search (index disabled)
 Limit (actual time=9.420..9.421 rows=5.00 loops=1)
   ->  Sort (actual time=9.420..9.420 rows=5.00 loops=1)
         Sort Method: top-N heapsort  Memory: 25kB
         ->  Seq Scan on docs_vec (actual time=0.004..7.922 rows=50000.00 loops=1)
 Execution Time: 9.428 ms
```

Both returned the same five IDs (123, 42459, 12281, 41406, 30268) in this run. HNSW is
**approximate**: on real data and larger tables it can miss some true neighbours, trading recall for
speed, tunable with `hnsw.ef_search` at query time and `m` / `ef_construction` at build time. Measure
recall against exact search on your own data before relying on it. Filtering (`WHERE tenant_id = ...
ORDER BY embedding <=> ...`) interacts with approximate indexes in non-obvious ways; pgvector 0.8 added
iterative index scans to help. Check the project's documentation for the version you install — it is
evolving quickly.

## Other notable extensions (not run here)

| Extension | What it does |
|-----------|--------------|
| PostGIS | geographic types, spatial indexes and thousands of GIS functions — the reference spatial database |
| TimescaleDB | time-series hypertables, compression, continuous aggregates (licensing differs by feature) |
| pg_partman | automatic creation and retention of partitions (lesson 6) |
| pg_cron | cron-style job scheduling inside the database |
| pg_repack | rebuild bloated tables and indexes with only brief locks |
| pgaudit | detailed session and object audit logging for compliance |

Before adopting any third-party extension, check that it supports your PostgreSQL major version,
that your hosting provider allows it, and how upgrades will work: an extension that lags behind a
new PostgreSQL release can block your upgrade.

## How It Actually Works

An extension is a **control file** (`name.control`: default version, whether it is trusted or
relocatable, required libraries) plus versioned **SQL scripts** (`name--1.0.sql`, and upgrade scripts
`name--1.0--1.1.sql`), usually alongside a compiled shared library. `CREATE EXTENSION` runs the
install script inside a transaction and records each created object in `pg_depend` with a dependency
of type "extension member". `ALTER EXTENSION UPDATE` finds a path through the upgrade scripts from
the installed version to the target.

C extensions are loaded into backend processes with `dlopen` when first needed, or into every
process at start-up via `shared_preload_libraries` when they must allocate shared memory, register
background workers or install hooks (the planner, executor and many other subsystems expose function
pointers extensions can wrap — that is how `pg_stat_statements` and `auto_explain` observe every
query). Index access methods like HNSW plug into the same interface as built-in GIN and GiST.

Because C extensions run inside the server process, a bug in one can crash the server: trust them as
you trust the PostgreSQL binary itself.

## Common mistakes

- Creating an extension in one database and expecting it in another.
- Upgrading the package but forgetting `ALTER EXTENSION ... UPDATE` in each database.
- Unbounded foreign-table queries where nothing was pushed down.
- Treating approximate vector search results as exact.
- Adopting an extension your managed provider or upgrade path does not support.

## Exercise

1. List the extensions available on your server and those created in each database. Which are
   outdated relative to `default_version`?
2. Create two databases, connect them with `postgres_fdw`, and join a local and a foreign table.
   Use `EXPLAIN VERBOSE` to find a query whose `WHERE` clause is *not* pushed down, and rewrite it so
   it is.
3. After a restart, use `pg_buffercache` to see how empty the cache is; run your workload; check again.
   Then try `pg_prewarm` on your largest hot table.
4. If you can install pgvector, build an HNSW index on 100,000 vectors, compute recall@10 against exact
   search for 100 random queries at two `hnsw.ef_search` settings, and record the latency trade-off.
