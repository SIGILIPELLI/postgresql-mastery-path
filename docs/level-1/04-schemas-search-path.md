---
description: "Organize a PostgreSQL database with schemas, understand how search_path resolves names, avoid the public-schema and function-shadowing traps."
---

# 04 · Schemas, search_path & Organizing Objects

A schema is a namespace inside a database. It is the tool PostgreSQL gives you for keeping a large
database navigable: billing tables in one place, reporting views in another, an extension's
functions out of everyone's way. Schemas also carry permissions, which makes them the natural unit
for "this team can create things here, that team can only read".

The part people get wrong is not creating schemas — it is **name resolution**. Which `invoices`
does `SELECT * FROM invoices` read? That is decided by `search_path`, and misunderstanding it causes
everything from "table not found" confusion to real security holes.

## Creating and using schemas

```sql
CREATE SCHEMA billing;
CREATE SCHEMA reporting;

CREATE TABLE billing.invoices (id int PRIMARY KEY, total numeric);
CREATE TABLE public.invoices  (id int PRIMARY KEY, note text);
```

Two tables with the same name, in different schemas, in one database — perfectly legal. A
**qualified** name (`billing.invoices`) is unambiguous. An unqualified name goes through the search
path.

## How `search_path` resolves names

```text
SHOW search_path;
   search_path
-----------------
 "$user", public

SELECT current_schemas(true);
   current_schemas
---------------------
 {pg_catalog,public}
```

The default path is `"$user", public`: first a schema with the same name as the current role, if
one exists, then `public`. `current_schemas(true)` reveals the full effective path, including
`pg_catalog` — the schema holding built-in tables, types and functions — which is searched
**first** unless you list it explicitly somewhere else. Schemas that do not exist (like `postgres`
here, the `$user` entry) are skipped silently.

Change the path and the same unqualified name means something else:

```sql
SET search_path = billing, public;
\d invoices
```

```text
             Table "billing.invoices"
 Column |  Type   | Collation | Nullable | Default
--------+---------+-----------+----------+---------
 id     | integer |           | not null |
 total  | numeric |           |          |
```

The first schema in the path is also where **unqualified `CREATE` statements put new objects**:

```sql
CREATE TABLE where_am_i (x int);
SELECT relnamespace::regnamespace FROM pg_class WHERE relname = 'where_am_i';
```

```text
 relnamespace
--------------
 billing
```

That is a common source of "I created the table but the app can't see it": the migration ran with
a different `search_path` from the application.

## Setting the path for real

`SET search_path` lasts for the session. Durable options, from narrowest to widest:

```sql
ALTER ROLE app_user SET search_path = app_user, billing;      -- every session of this role
ALTER DATABASE shop SET search_path = billing, public;         -- every session in this database
ALTER ROLE app_user IN DATABASE shop SET search_path = billing; -- both
-- or search_path in postgresql.conf for the whole cluster
```

Many teams skip all of this and **always schema-qualify** names in application SQL and migrations.
It is more typing and completely unambiguous.

## The public schema since PostgreSQL 15

Before PostgreSQL 15, every role could create objects in `public`. That changed:

```sql
CREATE ROLE app_user LOGIN;
SET ROLE app_user;
CREATE TABLE public.sneaky (x int);
```

```text
ERROR:  permission denied for schema public
```

```text
SELECT nspname, nspacl FROM pg_namespace WHERE nspname = 'public';
 nspname |                            nspacl
---------+---------------------------------------------------------------
 public  | {pg_database_owner=UC/pg_database_owner,=U/pg_database_owner}
```

Read that ACL as: the database owner has `U`sage and `C`reate; everybody (`=` with an empty
grantee means PUBLIC) has only `U`sage. Databases upgraded from older versions with `pg_upgrade` or
restored from old dumps may still have the old, permissive ACL. Check yours, and if you see
`=UC/postgres`, run `REVOKE CREATE ON SCHEMA public FROM PUBLIC;`.

## The per-user schema pattern

Because `"$user"` is first in the default path, giving a role a schema named after it gives that
role a private workspace with no configuration:

```sql
CREATE SCHEMA app_user AUTHORIZATION app_user;
SET ROLE app_user;
CREATE TABLE mine (x int);
SELECT relnamespace::regnamespace FROM pg_class WHERE relname = 'mine';
```

```text
 relnamespace
--------------
 app_user
```

This is handy for analysts' scratch tables in a shared warehouse.

## Function shadowing: why the path is a security setting

Here is a demonstration of why the order of `search_path` matters. Create a function named like a
built-in:

```sql
CREATE FUNCTION public.lower(text) RETURNS text LANGUAGE sql AS $$ SELECT 'hijacked' $$;
SELECT lower('ABC');
```

```text
 which_lower
-------------
 abc
```

Safe — `pg_catalog` is implicitly searched first. Now put `pg_catalog` after `public` explicitly:

```sql
SET search_path = public, pg_catalog;
SELECT lower('ABC');
```

```text
 which_lower_now
-----------------
 hijacked
```

Anyone who can create objects in a schema that appears in your path *before* the schema you meant
can make your queries call their code. With `public` locked down (previous section) this is much
harder, but the same rule applies to every schema writable by untrusted roles. It matters most for
`SECURITY DEFINER` functions (Level 3 · 03 and Level 4 · 08), which run with their owner's
privileges: always give them `SET search_path = pg_catalog, pg_temp` or a fixed safe list.

Also note that function resolution considers argument types: if an attacker's function is a
*better type match* than the built-in, it can win even when `pg_catalog` is first. Schema-qualifying
critical calls in privileged code (`pg_catalog.lower(x)`) removes the ambiguity entirely.

## A practical layout

A typical layout for a single-product application:

| Schema | Contains | Who can create |
|--------|----------|----------------|
| `app` | application tables, owned by a migration role | migration role only |
| `api` | views/functions exposed to the app or a REST layer | migration role |
| `reporting` | materialized views for dashboards | analytics role |
| `extensions` | extension objects (`CREATE EXTENSION pg_trgm SCHEMA extensions`) | superuser/owner |
| `audit` | audit-log tables written by triggers | nobody but triggers |

And an application role whose `search_path` lists exactly the schemas it uses.

## Moving and renaming

```sql
ALTER TABLE public.invoices SET SCHEMA reporting;   -- move (indexes and sequences follow)
ALTER SCHEMA reporting RENAME TO analytics;
DROP SCHEMA analytics;                              -- fails if not empty
DROP SCHEMA analytics CASCADE;                      -- drops everything inside — be careful
```

These are catalog-only operations: no data is copied, so they are instant even for huge tables.
Code that referred to the old qualified name will break, though, and views referencing the table
keep working because they store OIDs, not names.

## How It Actually Works

Every object in a database lives in the `pg_namespace` catalog's row for its schema: `pg_class` (tables,
indexes, views, sequences) has a `relnamespace` column, `pg_proc` (functions) has `pronamespace`, and
so on. A schema is nothing more than that row plus its ACL.

When the parser meets an unqualified name, it walks the **effective** search path — `pg_temp` (your
temporary schema, if you have created temp objects) first for tables, then `pg_catalog` unless it
was placed explicitly, then the configured entries — and takes the first match. Functions and
operators are different: PostgreSQL collects *every* candidate with that name across the path and
then picks the best match on argument types, using path position only to break ties. That is why
type-based shadowing is possible.

Once a statement is parsed, references are stored as OIDs. Views and functions written in SQL with
`BEGIN ATOMIC` bodies are bound to objects at creation time, so renaming or moving a table does not
break them. Functions written in PL/pgSQL or with string bodies are *re-parsed* at run time with the
caller's `search_path` — which is why those are the ones that need an explicit `SET search_path`.

## Common mistakes

- Migrations and the application running with different `search_path` values, so tables are created
  in one schema and queried in another.
- Assuming `public` is writable (it no longer is by default) — or assuming it is locked down on a
  database restored from an old dump.
- Putting user-writable schemas ahead of trusted ones in the path.
- `DROP SCHEMA ... CASCADE` in a script, run against the wrong database.
- Using separate *databases* for modules that need to join each other's data. Use schemas.

## Exercise

1. In your lab database, create schemas `app`, `reporting` and `extensions`. Create the `pg_trgm`
   extension in `extensions`.
2. Create a role `web` and configure its `search_path` so that unqualified names find `app` first,
   then `extensions`. Connect as `web` and confirm with `current_schemas(true)`.
3. Create the same table name in `app` and `public`. Predict which one `SELECT` reads for `web` and for
   `postgres`, then check.
4. Check the ACL on `public` in your database. Is it the PostgreSQL 15+ default?
5. Reproduce the `lower()` shadowing demo, then explain in two sentences how locking down `public`
   prevents it.
