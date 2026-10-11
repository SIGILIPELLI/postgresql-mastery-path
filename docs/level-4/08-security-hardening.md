---
description: "Harden PostgreSQL: TLS with verify-full, strict pg_hba.conf, revoking PUBLIC defaults, SECURITY DEFINER checks, audits and patching."
---

# 08 · Security Hardening

Earlier lessons covered pieces of PostgreSQL security as they came up: roles and default privileges
(Level 1 · 05), `search_path` hijacking (L1 · 04), SQL injection in dynamic SQL (L3 · 03), row-level
security (L3 · 07). This lesson assembles a hardening pass for a whole server: encrypt connections and
verify who you are talking to, tighten defaults that are permissive out of the box, and audit for the
mistakes that accumulate over time. Everything was run against this course's lab cluster on
PostgreSQL 18.6 — which, as you will see, fails the audit in instructive ways.

This is defensive configuration of your own server. It complements, rather than replaces, the
application-level practices covered in the [Cybersecurity Mastery
Path](https://sigilipelli.github.io/cybersecurity-mastery-path/).

## 1. Network exposure

- `listen_addresses` — only the interfaces clients need; never `'*'` on a host with a public interface
  unless a firewall restricts the port.
- Put the database on a private network; reach it through the application tier, a bastion or a VPN.
- Firewall port 5432 to known client addresses even inside the private network.

## 2. Encrypt connections with TLS — and verify the server

A test certificate authority and a server certificate for `localhost` (lab only; production
certificates come from your internal CA or a certificate service):

```bash
openssl req -new -x509 -days 30 -nodes -newkey rsa:2048 -subj "/CN=Lab Root CA" -keyout ca.key -out root.crt
openssl req -new -nodes -newkey rsa:2048 -subj "/CN=localhost" -keyout server.key -out server.csr
printf "subjectAltName=DNS:localhost,IP:127.0.0.1\n" > san.ext
openssl x509 -req -in server.csr -CA root.crt -CAkey ca.key -CAcreateserial -days 30 -extfile san.ext -out server.crt
chmod 600 server.key          # the server refuses a key readable by others
```

With `server.crt` and `server.key` in the data directory (the default file names), enable TLS — a reload
is enough:

```sql
ALTER SYSTEM SET ssl = on;
SELECT pg_reload_conf();
```

```text
$ psql "host=localhost ... sslmode=require" -Atc "select ssl, version, cipher from pg_stat_ssl where pid = pg_backend_pid()"
t|TLSv1.3|TLS_AES_256_GCM_SHA384
```

Encrypted. But `sslmode=require` only encrypts; it does **not** check that the server is who it claims
to be, so a man-in-the-middle with any certificate is accepted:

```text
$ psql "host=db.wrong-name.example hostaddr=127.0.0.1 ... sslmode=require" -Atc "select 'require does not check the name'"
require does not check the name
```

`sslmode=verify-full` checks the certificate chain **and** the host name:

```text
$ psql "host=localhost ... sslmode=verify-full"
psql: error: ... root certificate file "/Users/.../.postgresql/root.crt" does not exist
Either provide the file, use the system's trusted roots with sslrootcert=system, or change sslmode to disable server certificate verification.

$ psql "host=localhost ... sslmode=verify-full sslrootcert=root.crt" -Atc "select 'verified', ssl from pg_stat_ssl where pid = pg_backend_pid()"
verified|t

$ psql "host=db.wrong-name.example hostaddr=127.0.0.1 ... sslmode=verify-full sslrootcert=root.crt"
psql: error: connection to server at "127.0.0.1", port 54329 failed: server certificate for "localhost" (and 1 other name) does not match host name "db.wrong-name.example"
```

Clients should use `verify-full` with the CA certificate distributed to them. On the server side, require
TLS for remote connections in `pg_hba.conf` with `hostssl` lines (and `hostnossl ... reject` if you want to
be explicit). Set `ssl_min_protocol_version = 'TLSv1.2'` (the default) or higher. For machine clients,
client certificates (`cert` authentication, or `clientcert=verify-full` on a `scram-sha-256` line) add a
second factor.

## 3. pg_hba.conf: no trust, narrow rules

The audit query from Level 1 · 05, run against the lab:

```text
SELECT line_number, type, database, user_name, address, auth_method
FROM pg_hba_file_rules WHERE auth_method IN ('trust', 'password', 'md5') ORDER BY 1;
 line_number | type  |   database    | user_name |  address  | auth_method
-------------+-------+---------------+-----------+-----------+-------------
         117 | local | {all}         | {all}     |           | trust
         119 | host  | {all}         | {all}     | 127.0.0.1 | trust
         121 | host  | {all}         | {all}     | ::1       | trust
         124 | local | {replication} | {all}     |           | trust
         125 | host  | {replication} | {all}     | 127.0.0.1 | trust
         126 | host  | {replication} | {all}     | ::1       | trust
```

Every one of these lines would be a finding on a real server: anyone who can reach the socket or loopback
port can connect as any role, including `postgres`, with no password. A production `pg_hba.conf`:

```text
# TYPE   DATABASE      USER             ADDRESS          METHOD
local    all           postgres                          peer
hostssl  app           app_rw,app_ro    10.0.1.0/24      scram-sha-256
hostssl  replication   replicator       10.0.2.10/32     scram-sha-256
hostssl  all           +dba             10.0.9.0/24      scram-sha-256 clientcert=verify-full
# anything else: no match → rejected
```

`password` (cleartext) and `md5` (deprecated, Level 1 · 05) should not appear either.

## 4. Revoke permissive defaults

**CONNECT and TEMP on every database are granted to PUBLIC by default:**

```text
    datname    | public_connect | public_temp
---------------+----------------+-------------
 adv           | t              | t
 bench         | t              | t
 ...
 ticketing     | t              | t
```

So `app_user`, a role that has nothing to do with ticketing, can connect to it and create temporary tables
there:

```text
$ psql -U app_user -d ticketing -Atc "select 'app_user connected to ticketing'"
app_user connected to ticketing
```

```sql
REVOKE CONNECT, TEMP ON DATABASE ticketing FROM PUBLIC;
GRANT CONNECT ON DATABASE ticketing TO ticketing_app, ticketing_report;
```

```text
$ psql -U app_user -d ticketing -Atc "select 1"
psql: error: ... FATAL:  permission denied for database "ticketing"
DETAIL:  User does not have CONNECT privilege.

$ psql -U ticketing_app -d ticketing -Atc "select 'ticketing_app still connects'"
ticketing_app still connects
```

Also check:

- `CREATE` on schema `public` (revoked from PUBLIC by default since PostgreSQL 15; check databases
  restored from older dumps).
- `EXECUTE` on functions is granted to PUBLIC by default — revoke it on anything sensitive, and set
  `ALTER DEFAULT PRIVILEGES REVOKE EXECUTE ON FUNCTIONS FROM PUBLIC` for the owner role.

## 5. Audit privileges regularly

Run checks like these on a schedule and treat new rows as findings:

```sql
-- powerful roles
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb, rolbypassrls, rolreplication FROM pg_roles
WHERE (rolsuper OR rolcreaterole OR rolbypassrls OR rolreplication) AND rolname NOT LIKE 'pg\_%';
```

```text
  rolname   | rolsuper | rolcreaterole | rolcreatedb | rolbypassrls | rolreplication
 postgres   | t        | t             | t           | t            | t
 replicator | f        | f             | f           | f            | t
```

```sql
-- login roles with no password at all (acceptable only with peer or cert authentication)
SELECT rolname FROM pg_authid WHERE rolcanlogin AND rolpassword IS NULL;
```

```text
 app_user
 postgres
```

```sql
-- SECURITY DEFINER functions without a pinned search_path (Level 3 · 03)
SELECT p.oid::regprocedure AS function, r.rolname AS owner, p.proconfig
FROM pg_proc p JOIN pg_roles r ON r.oid = p.proowner
WHERE p.prosecdef
  AND NOT EXISTS (SELECT 1 FROM unnest(p.proconfig) c WHERE c LIKE 'search_path=%');
```

The lab's databases were clean (the Level 3 project's audit function pins its path). A deliberately bad
function shows what a finding looks like:

```text
CREATE FUNCTION public.risky_definer() RETURNS int LANGUAGE sql SECURITY DEFINER AS 'SELECT 1';
    function     |  owner   | proconfig
-----------------+----------+-----------
 risky_definer() | postgres |
```

A `SECURITY DEFINER` function owned by a superuser and callable by PUBLIC is a privilege-escalation
candidate. Also audit: who has the `pg_read_server_files`, `pg_write_server_files` and
`pg_execute_server_program` predefined roles (superuser-equivalent in practice), and which roles own
objects they should not.

## 6. Superuser hygiene

- Applications never connect as superuser or as table owners (Level 1 · 05, Level 3 · 07).
- Humans use personal roles with the privileges they need, `SET ROLE` to an owner role for migrations, and
  use superuser only for genuine administration — so the logs say who did what.
- Managed services do not give you a true superuser at all; design so you do not need one.

## 7. Logging and auditing

```text
log_connections = 'receipt,authentication,authorization'   # PostgreSQL 18 accepts a list of aspects
log_disconnections = on
log_statement = 'ddl'                                      # every schema change, with user and time
log_line_prefix = '%m [%p] %q%u@%d from %h '
```

For compliance-grade auditing of reads and writes per object, the `pgaudit` extension is the standard tool
(not run here). Ship logs off the server; an attacker with superuser can edit local log files.

## 8. Data protection

- PostgreSQL has no built-in transparent data encryption in the community release; encrypt at the storage
  layer (encrypted volumes or disks) and make sure **backups and WAL archives are encrypted too** — they
  contain everything.
- Column-level encryption (`pgcrypto`, Level 3 · 08) protects specific fields but moves the problem to key
  management.
- Mask or synthesise data when copying production into lower environments.

## 9. Patching

Minor releases (18.5 → 18.6) contain security and data-corruption fixes and do not change the on-disk
format: install them promptly — a restart is all it takes. The `\restrict` lines that `pg_dump` now emits
(Level 1 · 09) are an example of a security fix that arrived in a minor release. Track the project's
security announcements, and plan major upgrades before a version reaches end of life (lesson 9).

## How It Actually Works

When a client connects, the backend first negotiates TLS if requested (`SSLRequest`, or since PostgreSQL 17
an optional direct TLS handshake), then evaluates `pg_hba.conf` top to bottom using the connection type
(`hostssl` only matches encrypted connections), database, user and address, and runs the chosen
authentication method. Only after authentication does it check the role's `CONNECT` privilege on the
database — which is why the revoked `app_user` received "permission denied for database" rather than an
authentication error.

Client-side verification is entirely the client's job: libpq's `verify-ca` checks the chain against
`sslrootcert`, and `verify-full` additionally compares the requested host name against the certificate's
subject alternative names (or common name). Nothing on the server can force a client to verify — so the
connection strings your applications use are part of your security configuration.

## Common mistakes

- `sslmode=require` and believing the connection is protected against interception.
- `trust` lines left from setup or debugging.
- Leaving PUBLIC's default `CONNECT`, `TEMP` and function `EXECUTE` privileges in place.
- Superuser-owned `SECURITY DEFINER` functions without a pinned `search_path`.
- Unencrypted backups and WAL archives of an encrypted database.
- Delaying minor-release updates.

## Exercise

1. Create your own test CA and server certificate, enable TLS, and change `pg_hba.conf` so that TCP
   connections require `hostssl` with `scram-sha-256`. Prove a non-TLS connection is rejected and a
   `verify-full` one succeeds.
2. Run all of this lesson's audit queries on your server and fix every finding.
3. Revoke `CONNECT` from PUBLIC on every application database and grant it only to the roles that need it.
   Verify with `has_database_privilege`.
4. Enable `log_statement = 'ddl'` and connection logging, perform a migration, and reconstruct from the log
   who changed what and from where.
