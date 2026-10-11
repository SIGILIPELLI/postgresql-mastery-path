---
description: "Capstone: a hardened PostgreSQL primary and replica with TLS, roles, archiving and health checks, proven by PITR and failover drills."
---

# 10 · Capstone — Production-Ready PostgreSQL

Every earlier lesson examined one aspect of running PostgreSQL. This capstone assembles them into a
single deployment and then **proves** it works by breaking it: an accidental mass `DELETE` recovered
by point-in-time restore, and a primary failure handled by promoting the replica.

Everything ran on PostgreSQL 18.6 on one laptop — a primary on port 55432 and a replica on 55433. That is
the shape of a production deployment, not a production deployment: real servers go on separate hosts
in separate failure zones, certificates come from a real CA, and archives go to remote storage. Those
substitutions are noted where they matter.

## What "production-ready" means here

| Requirement | Implemented by | Lesson |
|-------------|----------------|--------|
| Only encrypted, password-verified connections | `hostssl` + `scram-sha-256` only; clients use `verify-full` | 4 · 08 |
| Least privilege | owner / read-write / read-only / monitor / replication roles; no PUBLIC connect | 1 · 05, 4 · 08 |
| Guard rails | per-role `statement_timeout`, `lock_timeout`, read-only role, idle-transaction timeout | 4 · 01 |
| Observability | `pg_stat_statements`, slow-query and DDL logging, a health-check script | 4 · 06 |
| Recoverability | WAL archiving + verified base backups, a timed PITR drill | 4 · 05 |
| Availability | streaming replica over TLS with a capped slot, a failover drill | 4 · 03 |

## Step 1 — the primary

```bash
#!/usr/bin/env bash
# Build a production-shaped primary on this machine (lab ports 55432/55433)
set -euo pipefail
export LC_ALL=en_US.UTF-8
BIN=/opt/homebrew/opt/postgresql@18/bin
BASE=$PWD
PGDATA=$BASE/primary
ARCHIVE=$BASE/wal_archive
mkdir -p "$ARCHIVE"

# 1. cluster with SCRAM from the first moment (superuser password from a file, not the command line)
printf '%s\n' "$(openssl rand -hex 16)" > "$BASE/.superuser_pw"; chmod 600 "$BASE/.superuser_pw"
"$BIN/initdb" -D "$PGDATA" -U postgres --pwfile="$BASE/.superuser_pw" \
  --auth-local=scram-sha-256 --auth-host=scram-sha-256 -E UTF8 --locale=en_US.UTF-8 > /dev/null

# 2. TLS certificate (lab CA; production uses your real CA)
openssl req -new -x509 -days 30 -nodes -newkey rsa:2048 -subj "/CN=Capstone CA" -keyout "$BASE/ca.key" -out "$BASE/root.crt" 2>/dev/null
openssl req -new -nodes -newkey rsa:2048 -subj "/CN=localhost" -keyout "$PGDATA/server.key" -out "$BASE/server.csr" 2>/dev/null
printf 'subjectAltName=DNS:localhost,IP:127.0.0.1\n' > "$BASE/san.ext"
openssl x509 -req -in "$BASE/server.csr" -CA "$BASE/root.crt" -CAkey "$BASE/ca.key" -CAcreateserial -days 30 \
  -extfile "$BASE/san.ext" -out "$PGDATA/server.crt" 2>/dev/null
chmod 600 "$PGDATA/server.key"

# 3. configuration: one include file under version control
cat > "$PGDATA/conf.d.prod.conf" <<CONF
listen_addresses = 'localhost'
port = 55432
unix_socket_directories = ''
max_connections = 100
ssl = on
password_encryption = scram-sha-256
shared_buffers = '512MB'
effective_cache_size = '4GB'
work_mem = '16MB'
maintenance_work_mem = '256MB'
random_page_cost = 1.1
wal_level = replica
max_wal_size = '2GB'
wal_compression = lz4
checkpoint_timeout = '15min'
archive_mode = on
archive_command = 'test ! -f $ARCHIVE/%f && cp %p $ARCHIVE/%f'
archive_timeout = '60s'
max_slot_wal_keep_size = '2GB'
shared_preload_libraries = 'pg_stat_statements'
log_min_duration_statement = '250ms'
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = '10s'
log_statement = 'ddl'
log_connections = 'authorization'
log_line_prefix = '%m [%p] %q%u@%d from %h '
idle_in_transaction_session_timeout = '5min'
CONF
echo "include 'conf.d.prod.conf'" >> "$PGDATA/postgresql.conf"

# 4. pg_hba: TLS + SCRAM only, nothing else
cat > "$PGDATA/pg_hba.conf" <<HBA
# TYPE   DATABASE     USER          ADDRESS          METHOD
hostssl  shopdb       +app_login    127.0.0.1/32     scram-sha-256
hostssl  shopdb       +app_login    ::1/128          scram-sha-256
hostssl  all          postgres      127.0.0.1/32     scram-sha-256
hostssl  all          postgres      ::1/128          scram-sha-256
hostssl  replication  replicator    127.0.0.1/32     scram-sha-256
hostssl  replication  replicator    ::1/128          scram-sha-256
HBA
"$BIN/pg_ctl" -D "$PGDATA" -l "$BASE/primary.log" start > /dev/null
echo "primary started"
```

Points worth noticing:

- The superuser password comes from a file (`--pwfile`), never the command line or shell history, and
  `initdb` sets SCRAM for local and host connections from the start — there is never a `trust` window.
- Configuration lives in one included file, `conf.d.prod.conf`, which is what you would keep in version
  control; `postgresql.conf` stays the stock file plus one `include` line.
- `max_slot_wal_keep_size = '2GB'` caps what an abandoned replica slot can hold (lesson 3), and
  `wal_compression = lz4` shrinks full-page images (Level 2 · 09).
- `pg_hba.conf` is replaced entirely. Six lines, all `hostssl`, all `scram-sha-256`; anything that does
  not match is rejected.

## Step 2 — roles, database and schema

```sql
\set ON_ERROR_STOP on
-- roles: an owner nobody logs in as, capability groups, and login roles
CREATE ROLE shop_owner NOLOGIN;
CREATE ROLE app_login NOLOGIN;                  -- pg_hba group: who may log in to shopdb
CREATE ROLE shop_rw NOLOGIN;
CREATE ROLE shop_ro NOLOGIN;
CREATE ROLE shop_app LOGIN IN ROLE shop_rw, app_login;
CREATE ROLE shop_readonly LOGIN IN ROLE shop_ro, app_login;
CREATE ROLE monitor LOGIN IN ROLE pg_monitor, app_login;
CREATE ROLE replicator LOGIN REPLICATION;
\password shop_app
\password shop_readonly
\password monitor
\password replicator
ALTER ROLE shop_app SET statement_timeout = '15s';
ALTER ROLE shop_app SET lock_timeout = '5s';
ALTER ROLE shop_readonly SET statement_timeout = '5min';
ALTER ROLE shop_readonly SET default_transaction_read_only = on;

CREATE DATABASE shopdb OWNER shop_owner;
REVOKE CONNECT, TEMP ON DATABASE shopdb FROM PUBLIC;
GRANT CONNECT ON DATABASE shopdb TO shop_rw, shop_ro, monitor;
\c shopdb
CREATE EXTENSION pg_stat_statements;
SET ROLE shop_owner;
CREATE SCHEMA shop;
GRANT USAGE ON SCHEMA shop TO shop_rw, shop_ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA shop GRANT SELECT ON TABLES TO shop_ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA shop GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO shop_rw;
ALTER DEFAULT PRIVILEGES IN SCHEMA shop REVOKE EXECUTE ON FUNCTIONS FROM PUBLIC;
CREATE TABLE shop.customers (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email text NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now());
CREATE UNIQUE INDEX customers_email_lower ON shop.customers (lower(email));
CREATE TABLE shop.orders (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES shop.customers,
  total numeric(12,2) NOT NULL CHECK (total >= 0),
  status text NOT NULL DEFAULT 'new' CHECK (status IN ('new','paid','shipped','cancelled')),
  created_at timestamptz NOT NULL DEFAULT now());
CREATE INDEX ON shop.orders (customer_id, created_at DESC);
RESET ROLE;
```

Run as the superuser, with `\password` hashing each password on the client (Level 1 · 05); I generated
the passwords with `openssl rand` and stored them in a `0600` `.pgpass` file that every later command
uses.

### Proving the access rules

As the application role, over verified TLS:

```text
INSERT 0 5000
INSERT 0 50000
 current_user | ssl
--------------+-----
 shop_app     | t
```

And everything that should fail, does:

```text
-- the app cannot change the schema
ERROR:  must be owner of table orders

-- the read-only role can read, not write
 count
 50000
ERROR:  cannot execute DELETE in a read-only transaction

-- the app cannot reach other databases
FATAL:  no pg_hba.conf entry for host "::1", user "shop_app", database "postgres", SSL encryption

-- no unencrypted connections at all
FATAL:  no pg_hba.conf entry for host "::1", user "shop_app", database "shopdb", no encryption

-- wrong password
FATAL:  password authentication failed for user "shop_app"
```

## Step 3 — the replica

```bash
#!/usr/bin/env bash
set -euo pipefail
export LC_ALL=en_US.UTF-8 PGPASSFILE=$PWD/.pgpass PGSSLMODE=verify-full PGSSLROOTCERT=$PWD/root.crt
BIN=/opt/homebrew/opt/postgresql@18/bin
"$BIN/psql" -X -h localhost -p 55432 -U postgres -d postgres -qc "SELECT pg_create_physical_replication_slot('replica1')"
"$BIN/pg_basebackup" -h localhost -p 55432 -U replicator -D replica -R -S replica1 -X stream -c fast
cat >> replica/conf.d.prod.conf <<CONF
port = 55433
shared_buffers = '256MB'
hot_standby_feedback = on
CONF
"$BIN/pg_ctl" -D replica -l replica.log start > /dev/null
```

```text
LOG:  database system is ready to accept read-only connections
LOG:  started streaming WAL from primary at 0/4000000 on timeline 1
```

Because the environment set `PGSSLMODE=verify-full` and `PGSSLROOTCERT`, `pg_basebackup -R` wrote those
into the replica's `primary_conninfo` — the replication stream itself verifies the primary's certificate:

```text
'user=replicator
host=localhost
port=55432
sslmode=''verify-full''
sslrootcert=''.../cap/root.crt''
```

## Step 4 — health checks

One query that returns a row per check, flagging problems — the alerting list from lesson 6 in runnable
form. It runs as the `monitor` role (a member of `pg_monitor`), never as a superuser:

```sql
-- 04_health_check.sql — run as a pg_monitor role; every row returned is a finding
WITH checks AS (
  SELECT 'role' AS item, CASE WHEN pg_is_in_recovery() THEN 'replica' ELSE 'primary' END AS detail, false AS problem
  UNION ALL
  SELECT 'connections', count(*) || ' of ' || current_setting('max_connections'),
         count(*) > 0.8 * current_setting('max_connections')::int
  FROM pg_stat_activity
  UNION ALL
  SELECT 'long transaction', pid || ' open for ' || date_trunc('second', now() - xact_start), true
  FROM pg_stat_activity WHERE xact_start < now() - interval '5 minutes' AND pid <> pg_backend_pid()
  UNION ALL
  SELECT 'blocked session', pid || ' blocked by ' || pg_blocking_pids(pid)::text, true
  FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0 AND now() - query_start > interval '30 seconds'
  UNION ALL
  SELECT 'replication', application_name || ' ' || state || ' lag ' ||
         pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) || ' / ' || coalesce(replay_lag::text, '0'),
         state <> 'streaming' OR pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) > 256 * 1024 * 1024
  FROM pg_stat_replication
  UNION ALL
  SELECT 'slot ' || slot_name, CASE WHEN active THEN 'active' ELSE 'INACTIVE' END || ', retains ' ||
         pg_size_pretty(pg_wal_lsn_diff(CASE WHEN pg_is_in_recovery() THEN pg_last_wal_replay_lsn() ELSE pg_current_wal_lsn() END, restart_lsn)) ||
         ', wal_status ' || wal_status,
         NOT active OR wal_status IN ('unreserved', 'lost')
  FROM pg_replication_slots
  UNION ALL
  SELECT 'archiver', archived_count || ' archived, ' || failed_count || ' failed, last ' ||
         coalesce(date_trunc('second', now() - last_archived_time)::text || ' ago', 'never'),
         failed_count > 0 AND last_failed_time > coalesce(last_archived_time, '-infinity')
  FROM pg_stat_archiver WHERE NOT pg_is_in_recovery()
  UNION ALL
  SELECT 'xid age ' || datname, age(datfrozenxid)::text, age(datfrozenxid) > 500000000
  FROM pg_database WHERE datallowconn
)
SELECT item, detail, CASE WHEN problem THEN 'PROBLEM' ELSE 'ok' END AS status FROM checks ORDER BY problem DESC, item;
```

My first version named the first column `check` and failed with `syntax error at or near "check"` —
`CHECK` is a reserved word. Renamed to `item`, on the primary:

```text
       item        |                       detail                       | status
-------------------+----------------------------------------------------+--------
 archiver          | 4 archived, 0 failed, last 00:00:26 ago            | ok
 connections       | 11 of 100                                          | ok
 replication       | walreceiver streaming lag 0 bytes / 00:00:00.00062 | ok
 role              | primary                                            | ok
 slot replica1     | active, retains 0 bytes, wal_status reserved       | ok
 xid age postgres  | 38                                                 | ok
 xid age shopdb    | 38                                                 | ok
 xid age template1 | 38                                                 | ok
```

and on the replica:

```text
       item        |  detail  | status
-------------------+----------+--------
 connections       | 8 of 100 | ok
 role              | replica  | ok
 xid age postgres  | 38       | ok
 xid age shopdb    | 38       | ok
 xid age template1 | 38       | ok
```

(`pg_stat_activity` counts background processes too, which is why an idle server shows 8–11
"connections".) In production, run it every minute from your monitoring system and alert on any
`PROBLEM` row; add OS-level checks for disk space on the data and WAL volumes.

## Step 5 — the backup drill

**The first attempt failed, and the failure is the most useful result in this capstone.** I ran
`pg_basebackup -U postgres`:

```text
pg_basebackup: error: connection to server at "localhost" (::1), port 55432 failed: FATAL:  no pg_hba.conf entry for replication connection from host "::1", user "postgres", SSL encryption
```

Correct behaviour — the hardened `pg_hba.conf` only lets `replicator` make replication connections — but
my drill script did not stop on the error. It went on to run the "accident" (`DELETE FROM shop.orders
WHERE status = 'new'`, 50,001 rows) with **no base backup from before it**. Had this been real, that data
would have been recoverable only from the replica — which had already replayed the delete. Two lessons:
backup scripts must use `set -e` and alert on failure, and a backup you have not verified does not exist.

After reloading the test data, the drill again, correctly:

```bash
pg_basebackup -p 55432 -U replicator -D backup_base -X stream -c fast
pg_verifybackup backup_base
```

```text
backup successfully verified
safe point: 2026-10-11 13:06:30.499398+05:30
DELETE 50000
```

Restore into a separate directory and port, targeting the safe point (lesson 5):

```text
port = 55434
archive_mode = off
restore_command = 'cp .../wal_archive/%f %p'
recovery_target_time = '2026-10-11 13:06:30.499398+05:30'
recovery_target_action = 'promote'
```

```text
LOG:  recovery stopping before commit of transaction 789, time 2026-10-11 13:06:31.578009+05:30
LOG:  selected new timeline ID: 2

orders in drill copy: 50000
orders on damaged primary: 0
```

Every order recovered, in a copy that took about a second to start on this small database. On a real
system, time this drill at full size: that number is your recovery time objective. (One more practical
detail from the drill: the restore server's port was not in `.pgpass`, so the first verification query sat
at a password prompt in a non-interactive script until I killed it — use `psql -w` in scripts so they fail
instead of waiting.)

## Step 6 — the failover drill

The application connects with a **multi-host connection string** that only accepts a writable server:

```text
host=localhost,localhost port=55432,55433 dbname=shopdb user=shop_app target_session_attrs=read-write
```

```text
app writes to port 55432
primary stopped (fenced)
connection to server at "localhost" (::1), port 55433 failed: session is read-only
```

With the primary stopped — the "fence" — libpq tried the replica, found it read-only, and refused it, so
the application could not accidentally write to a standby. Then:

```bash
pg_ctl -D replica promote -w
```

```text
app writes to port 55433
order 100029 after failover
INSERT 0 1

LOG:  received promote request
LOG:  selected new timeline ID: 2
```

The same connection string now lands on the new primary with no application change. Note the order ID:
**100029**, not the next value after the primary's last order. Sequences are WAL-logged in batches of 32
values ahead of use; a promoted standby starts after the last logged value, so failovers leave a gap in
IDs — harmless, and one more reason never to depend on IDs being contiguous (Level 1 · 03).

Here every step was manual and on one machine. In production a tool such as Patroni (lesson 9) does the
fencing, promotion and reconfiguration, and the old primary must be rebuilt as a replica (`pg_rewind` or a
fresh clone) before you have redundancy again.

## The readiness checklist, as verified

| Check | Result |
|-------|--------|
| No `trust`, no non-TLS rules in `pg_hba.conf` | yes (six `hostssl` + `scram-sha-256` lines) |
| Clients and replication verify the server certificate | yes (`verify-full` in app and `primary_conninfo`) |
| Application cannot run DDL, reach other databases or connect unencrypted | proven by failed attempts |
| Per-role timeouts and a read-only reporting role | yes |
| `pg_stat_statements`, slow-query, lock-wait, temp-file and DDL logging | configured |
| WAL archiving working | `pg_stat_archiver`: archived, 0 failed |
| Base backup verified | `pg_verifybackup`: verified |
| PITR drill | recovered 50,000 rows to the second before the delete |
| Replica streaming with capped slot | lag 0 bytes; `max_slot_wal_keep_size = 2GB` |
| Failover drill | promoted; app reconnected via `target_session_attrs` |
| **Not done here** | separate hosts and zones, real CA, off-site encrypted archive, automated failover, disk-space alerts, load testing at real scale |

## How It Actually Works

Nothing in this capstone is new machinery; it is the earlier lessons' mechanisms composed. The security
layer is `pg_hba.conf` evaluation plus TLS negotiation plus role privileges (lessons 1 · 05 and 4 · 08).
Recoverability is the WAL stream used three ways: archived to files for PITR, streamed to the replica for
availability, and replayed by the startup process in both cases (Level 2 · 09). Observability reads the
shared-memory statistics and activity slots (lesson 6). The drills exercise the code paths you otherwise
only run during an incident — which is why they find problems like the one in step 5.

## Exercise

Build this deployment yourself — on three virtual machines or containers if you can, so the primary,
replica and backup storage are genuinely separate — and then:

1. Automate the whole build (the four scripts above, or Ansible/Terraform) so it is reproducible from
   nothing.
2. Schedule the health check and send `PROBLEM` rows to a real alert channel. Trigger each check
   deliberately (an idle transaction, a stopped replica, a broken `archive_command`) and confirm the alert.
3. Schedule nightly base backups with retention, and a weekly automated PITR drill that restores to a
   random recent time, runs sanity queries and reports the elapsed time.
4. Replace the manual failover with Patroni (or your platform's equivalent), run the failover drill again,
   and measure the real write outage from the application's point of view.
5. Write the one-page runbook for "the primary is down" and "someone deleted data", and have someone else
   follow it.
