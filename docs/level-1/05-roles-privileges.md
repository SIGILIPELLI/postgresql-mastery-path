---
description: "Build a least-privilege PostgreSQL setup: login and group roles, GRANT, default privileges, predefined roles, SCRAM passwords and pg_hba.conf rules."
---

# 05 · Roles, Privileges & Client Authentication

Two separate systems decide whether a query runs. **Authentication** — governed by `pg_hba.conf`
— decides whether a connection is allowed in at all and how it must prove its identity.
**Authorization** — roles and privileges — decides what an authenticated session may touch. Most
real incidents involve one of three failures: everything running as a superuser, privileges that
work today but not for the next table someone creates, and a `pg_hba.conf` line that is more
permissive than anyone realised. This lesson addresses all three.

## Roles: users and groups are the same thing

PostgreSQL has only **roles**. A role with the `LOGIN` attribute is what other systems call a user;
a role without it is a group. Roles are cluster-wide — the same `alice` exists in every database.

```sql
CREATE ROLE readers  NOLOGIN;
CREATE ROLE writers  NOLOGIN IN ROLE readers;     -- writers are also readers
CREATE ROLE alice    LOGIN PASSWORD 'alice-dev-pw' IN ROLE writers;
CREATE ROLE migrator LOGIN PASSWORD 'migrator-dev-pw';
```

```text
\du
                             List of roles
 Role name |                         Attributes
-----------+------------------------------------------------------------
 alice     |
 migrator  |
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS
 readers   | Cannot login
 writers   | Cannot login
```

(`CREATE USER` is just `CREATE ROLE ... LOGIN`.) Members **inherit** the privileges of the roles
they belong to by default, so `alice` gets everything granted to `writers` and to `readers`.

Attributes worth knowing: `SUPERUSER` (bypasses every check — reserve it for administration),
`CREATEDB`, `CREATEROLE`, `REPLICATION`, `BYPASSRLS` and `CONNECTION LIMIT n`.

## A least-privilege layout

The pattern used throughout this course:

- a **migration/owner role** that owns the schema and every table, and is used only by deploys;
- **group roles** describing capabilities (`readers`, `writers`);
- **login roles** for applications and people, which are members of groups and own nothing.

```sql
CREATE SCHEMA app AUTHORIZATION migrator;
GRANT USAGE ON SCHEMA app TO readers;

SET ROLE migrator;
CREATE TABLE app.orders (
  id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  total numeric NOT NULL
);
RESET ROLE;
```

Schema `USAGE` lets a role *look up* objects in the schema; it still needs privileges on each
object:

```text
SET ROLE alice;
SELECT * FROM app.orders;
ERROR:  permission denied for table orders
```

```sql
GRANT SELECT          ON ALL TABLES IN SCHEMA app TO readers;
GRANT INSERT, UPDATE  ON ALL TABLES IN SCHEMA app TO writers;
```

```text
SET ROLE alice;
INSERT INTO app.orders (total) VALUES (10) RETURNING id;
 id
----
  1
```

## The trap: "ALL TABLES" means "all tables that exist right now"

The migrator adds a table in the next release:

```sql
SET ROLE migrator;
CREATE TABLE app.refunds (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, order_id bigint);
```

```text
SET ROLE alice;
SELECT * FROM app.refunds;
ERROR:  permission denied for table refunds
```

`GRANT ... ON ALL TABLES` is a one-off loop over existing tables. The fix is **default privileges**,
which are attached to the *creating* role:

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE migrator IN SCHEMA app
  GRANT SELECT ON TABLES TO readers;
ALTER DEFAULT PRIVILEGES FOR ROLE migrator IN SCHEMA app
  GRANT INSERT, UPDATE ON TABLES TO writers;

SET ROLE migrator;
CREATE TABLE app.shipments (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY);
```

```text
SET ROLE alice;
SELECT count(*) FROM app.shipments;
 count
-------
     0
```

The important phrase is `FOR ROLE migrator`. Default privileges apply only to objects created *by
that role*. If someone runs a migration as `postgres` instead, the new table gets no grants — one
more reason all DDL should run as the owner role.

`\dp` shows the result. `refunds` was created before the defaults existed, so it still has none:

```text
\dp app.*
 Schema |       Name       |   Type   |     Access privileges      |
--------+------------------+----------+----------------------------+
 app    | orders           | table    | migrator=arwdDxtm/migrator+|
        |                  |          | readers=r/migrator        +|
        |                  |          | writers=aw/migrator        |
 app    | refunds          | table    |                            |
 app    | shipments        | table    | readers=r/migrator        +|
        |                  |          | writers=aw/migrator       +|
        |                  |          | migrator=arwdDxtm/migrator |
```

Decode the letters: `r` SELECT, `a` INSERT, `w` UPDATE, `d` DELETE, `D` TRUNCATE, `x` REFERENCES,
`t` TRIGGER, `m` MAINTAIN (new in PostgreSQL 17: allows VACUUM, ANALYZE, REINDEX and similar).
`readers=r/migrator` reads "readers has SELECT, granted by migrator". An empty column means "the
owner has everything, nobody else has anything". `\ddp` lists the default-privilege rules.

## Sequences: identity columns versus `serial`

`alice` could insert into `app.orders` without any grant on its sequence, because identity columns
use their sequence internally. A `serial` column is different — its default calls `nextval()`
with the caller's privileges:

```sql
SET ROLE migrator;
CREATE TABLE app.legacy (id serial PRIMARY KEY, v text);
GRANT INSERT ON app.legacy TO writers;
SET ROLE alice;
INSERT INTO app.legacy (v) VALUES ('x');
```

```text
ERROR:  permission denied for sequence legacy_id_seq
```

With `serial` you also need `GRANT USAGE ON SEQUENCE` (or a default privilege `ON SEQUENCES`).
Another small reason to prefer identity columns.

## Predefined roles

PostgreSQL ships roles for common needs, so you do not have to hand out superuser:

| Role | Grants |
|------|--------|
| `pg_read_all_data` / `pg_write_all_data` | read/write every table in every schema (14+) |
| `pg_monitor` | read monitoring views such as full `pg_stat_activity` |
| `pg_signal_backend` | cancel or terminate other sessions (not superusers') |
| `pg_read_server_files`, `pg_write_server_files`, `pg_execute_server_program` | server-side `COPY` to files/programs — effectively superuser-equivalent, grant with great care |
| `pg_maintain` | VACUUM, ANALYZE, REINDEX etc. on all relations (17+) |

```sql
GRANT pg_read_all_data TO readers;
SELECT pg_has_role('alice', 'pg_read_all_data', 'MEMBER');   -- t, through writers → readers
```

## Passwords

```text
SHOW password_encryption;
 scram-sha-256

SELECT rolname, left(rolpassword, 14) FROM pg_authid WHERE rolname = 'alice';
 alice   | SCRAM-SHA-256$
```

Passwords are stored as salted SCRAM-SHA-256 verifiers, never in plain text. Two details:

- `CREATE ROLE ... PASSWORD 'x'` sends the password in your SQL text, where it can end up in the
  server log (if statement logging is on) and in your shell history. psql's `\password alice`
  hashes on the client and sends only the verifier.
- MD5 password hashes are deprecated as of PostgreSQL 18 (the server warns when you set one).
  Clusters migrated from old versions may still hold MD5 hashes; resetting each password while
  `password_encryption = scram-sha-256` upgrades them.

## Client authentication: `pg_hba.conf`

`pg_hba.conf` is read **top to bottom, and the first matching line wins** — no fall-through.

```text
# TYPE  DATABASE  USER   ADDRESS        METHOD
host    shop      alice  127.0.0.1/32   scram-sha-256
host    shop      alice  ::1/128        scram-sha-256
local   all       all                   trust
host    all       all    127.0.0.1/32   trust
```

- **TYPE**: `local` (Unix socket), `host` (TCP, with or without SSL), `hostssl` (TCP with SSL only),
  `hostnossl`.
- **DATABASE**/**USER**: names, `all`, `+groupname` for members of a role, or `@file`.
- **METHOD**: `scram-sha-256`, `peer` (local socket: OS user must equal DB user), `cert`, `ldap`,
  `reject`, and `trust` (no check at all).

After editing, reload with `SELECT pg_reload_conf();` and verify what the server actually parsed —
syntax errors in the file show up here instead of silently breaking logins:

```text
SELECT line_number, type, database, user_name, address, auth_method
FROM pg_hba_file_rules ORDER BY line_number LIMIT 4;
 line_number | type  | database | user_name |  address  |  auth_method
-------------+-------+----------+-----------+-----------+---------------
           1 | host  | {shop}   | {alice}   | 127.0.0.1 | scram-sha-256
           2 | host  | {shop}   | {alice}   | ::1       | scram-sha-256
         119 | local | {all}    | {all}     |           | trust
         121 | host  | {all}    | {all}     | 127.0.0.1 | trust
```

Testing the rules on the lesson's cluster:

```text
$ psql -w -h 127.0.0.1 -U alice -d shop -c "select 1"
psql: error: connection to server at "127.0.0.1", port 54329 failed: fe_sendauth: no password supplied

$ PGPASSWORD=wrong psql -h 127.0.0.1 -U alice -d shop -c "select 1"
psql: error: connection to server at "127.0.0.1", port 54329 failed: FATAL:  password authentication failed for user "alice"

$ PGPASSWORD=alice-dev-pw psql -h 127.0.0.1 -U alice -d shop -Atc "select current_user, inet_client_addr()"
alice|127.0.0.1

$ psql -h 127.0.0.1 -U alice -d postgres -Atc "select current_user"
alice
```

The last line is the lesson: the scram rule only covered database `shop`, so a connection by alice
to database `postgres` fell through to the `trust` line and got in **without a password**. The server
log tells you which line a failed connection matched:

```text
FATAL:  password authentication failed for user "alice"
DETAIL:  Connection matched file "pg_hba.conf" line 1: "host    shop            alice           127.0.0.1/32            scram-sha-256"
```

On a real server, there should be no `trust` lines at all except perhaps `local` for the `postgres`
OS user via `peer`, and the last line should be a catch-all `reject` or simply nothing (no match
means rejected).

## How It Actually Works

Privileges are stored as **ACL arrays** on the objects themselves: `pg_class.relacl` for tables,
`pg_namespace.nspacl` for schemas, `pg_proc.proacl` for functions. A NULL ACL means "default":
owner has all privileges, PUBLIC has whatever the object type grants by default (for functions,
EXECUTE; for tables, nothing). Every permission check takes the current role's effective set —
itself plus every role it inherits from, computed by walking `pg_auth_members` — and looks for a
matching ACL entry. Owners and superusers short-circuit the check.

Default privileges are rows in `pg_default_acl`, keyed by creating role and schema. When a role
creates a table, PostgreSQL copies any matching rows into the new table's `relacl`. Nothing is
retroactive.

Authentication happens in the freshly forked backend before it touches any database: it reads the
pre-parsed `pg_hba.conf` rules (parsed by the postmaster at start and reload), finds the first
match on connection type, database, user and client address, then runs that method's protocol.
For SCRAM, the server sends a salt and iteration count, the client proves it knows the password
without ever sending it, and the server also proves to the client that it knows the verifier —
mutual authentication that MD5 never had.

## Common mistakes

- Applications connecting as `postgres` or as the table owner. An SQL injection then has
  superuser or DROP TABLE power.
- `GRANT ... ON ALL TABLES` without matching `ALTER DEFAULT PRIVILEGES` — works until the next
  migration.
- Running migrations as a different role than the one named in `FOR ROLE`.
- A broad `host all all 0.0.0.0/0 trust` "temporarily" left in `pg_hba.conf`.
- Not checking `pg_hba_file_rules` after an edit; a typo can lock everyone out on the next reload.
- Granting `pg_write_server_files` or `pg_execute_server_program` casually: they allow reading or
  writing arbitrary files as the server's OS user.

## Exercise

1. Create the owner/group/login role layout from this lesson in your lab database, with a
   `reporting` login that can only read.
2. As the owner role, create two tables. Verify `reporting` can read both without any further grants.
3. Create a table as `postgres` instead. What can `reporting` do with it? Fix it properly.
4. Add `pg_hba.conf` rules so that `reporting` must use a password over TCP for every database, then
   prove it with three connection attempts (no password, wrong password, right password).
5. Use `pg_hba_file_rules` to confirm your rules, then deliberately introduce a typo, reload, and see
   how the error is reported.
