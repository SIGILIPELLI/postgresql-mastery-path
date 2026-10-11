---
description: "Why PostgreSQL connections are expensive, what happens at max_connections, and PgBouncer pool modes, measured, with session-state traps."
---

# 02 · Connections & Pooling

Every PostgreSQL connection is a dedicated server process (Level 1 · 01). That design is robust, but
it makes connections expensive to open and expensive to keep: each holds memory, a slot in shared
data structures, and a share of CPU scheduling. Applications built on serverless functions, many
small services, or frameworks that open a pool per worker process can easily try to hold thousands of
connections — and PostgreSQL does not degrade gracefully when they do.

This lesson measures the problem and the standard fix, **PgBouncer**, on PostgreSQL 18.6 and
PgBouncer 1.26 (installed with Homebrew), using a 756 MB `pgbench` database.

## The cost of opening a connection

`pgbench -C` opens a new connection for every transaction — what an application without a pool does:

```text
--- direct, new connection per transaction
latency average = 8.023 ms
average connection time = 3.986 ms
tps = 997.129007 (including reconnection times)
```

Each query (a primary-key lookup taking a fraction of a millisecond) paid about 4 ms to connect: a
process fork, authentication, loading catalog caches for the database. Here that was over local TCP
with `trust` authentication; with TLS and SCRAM over a network, connection set-up costs more.

## The cost of keeping many connections

`max_connections` is 100 on this server. 200 clients:

```text
--- direct, 200 clients
pgbench: error: connection to server at "localhost" (::1), port 54329 failed: FATAL:  sorry, too many clients already
pgbench: error: could not create connection for client 162
```

Raising `max_connections` is a restart-only setting, and it is not free: some shared structures are
sized by it, and more processes compete for the same cores. Even below the limit, more connections do
not mean more throughput:

```text
--- direct, 10 clients    latency average = 0.119 ms   tps = 84379.458338
--- direct, 90 clients    latency average = 1.214 ms   tps = 74108.408791
```

On 8 cores, 90 concurrent clients delivered *less* throughput than 10, with ten times the latency per
query. Work beyond what the CPUs and disks can run in parallel just queues — inside the database, where
it holds locks and snapshots while waiting. A database typically performs best with active connections
in the order of a small multiple of its cores.

## Connection poolers

A pooler sits between clients and PostgreSQL, accepting many cheap client connections and multiplexing
them onto a small number of real server connections. Application-side pools (in your driver or
framework) do this within one process; **PgBouncer** does it for all clients in front of the database.

Minimal configuration used here:

```ini
[databases]
bench = host=127.0.0.1 port=54329 dbname=bench

[pgbouncer]
listen_addr = 127.0.0.1
listen_port = 6432
auth_type = trust                 ; lab only — use scram-sha-256 with auth_file or auth_query
auth_file = userlist.txt
pool_mode = transaction
default_pool_size = 10
max_client_conn = 500
max_prepared_statements = 100
admin_users = postgres
```

Clients connect to port 6432 instead of 5432; nothing else changes. The same two tests through it:

```text
--- pgbouncer, new client connection per transaction
latency average = 2.343 ms
average connection time = 1.159 ms
tps = 3414.960004 (including reconnection times)

--- via pgbouncer, 200 clients, 10 server connections
latency average = 3.201 ms
tps = 62472.777520 (without initial connection time)
```

- Connecting to PgBouncer (an event-driven process, no fork) took 1.2 ms instead of 4 ms, and
  throughput with connect-per-transaction more than tripled.
- 200 clients that the server rejected outright were served through 10 server connections. Each
  client's latency is higher because clients queue in PgBouncer for a free server connection — which
  is exactly where you want the queueing to happen, outside the database.

The admin console shows the pools: `psql -p 6432 -d pgbouncer -c "SHOW POOLS"` (columns
`cl_active`, `cl_waiting`, `sv_active`, `sv_idle`, …) — `cl_waiting` above zero for long periods means
the pool is too small or queries are too slow.

## Pool modes and their catch

| Mode | A server connection is assigned to a client... | Compatible with |
|------|----------------------------------------------|-----------------|
| `session` | for the whole client session | everything; little multiplexing |
| `transaction` | for one transaction, then returned | most apps, *if* they avoid session state |
| `statement` | for one statement (multi-statement transactions forbidden) | rarely used |

Transaction mode gives the big win and has one big rule: **nothing may depend on session state**,
because your next transaction may run on a different server connection — and someone else's next
transaction may run on yours. Two clients through a pool of size 1:

```text
A's backend pid: 66131
B's backend pid: 66131
B sees statement_timeout = 1ms
B's query was cancelled: canceling statement due to statement timeout
```

Client A ran a session-level `SET statement_timeout = '1ms'` and moved on. Client B, a different
application session, received the same server connection and inherited A's setting — its query was
cancelled for no reason it could see. The same leak affects `search_path`, the tenant settings from
Level 3 · 07 (a security problem there), temporary tables, session advisory locks, `LISTEN`, and
`WITH HOLD` cursors. Scope settings to the transaction instead:

```text
inside B's transaction: 5s          (SET LOCAL statement_timeout = '5s')
after it: 0
```

Prepared statements used to be the other big incompatibility. Since version 1.21, PgBouncer can track
protocol-level prepared statements in transaction mode when `max_prepared_statements` is set; psycopg
server-side prepares worked through it here:

```text
prepared via bouncer: (0,)
prepared via bouncer: (0,)
prepared via bouncer: (0,)
```

SQL-level `PREPARE` statements still do not survive transaction pooling.

## Sizing the pool

- Size **server-side** connections for the database (`default_pool_size` × databases × users, plus
  room for admin and replication), not for the number of clients.
- Start around 2–4 × CPU cores for OLTP and measure; more only helps if queries spend time waiting on
  I/O or locks.
- Keep application-side pools small too: 20 app instances × a pool of 50 is 1,000 connections before
  any PgBouncer.
- Put PgBouncer close to the clients or the database, and run more than one for availability (it is
  single-threaded; several instances can share a port with `so_reuseport` on Linux).

Managed services often include a pooler (and PostgreSQL-compatible proxies exist), with the same
transaction-mode rules.

## How It Actually Works

A PostgreSQL connection start runs: TCP accept by the postmaster, `fork()` of a new backend, client
authentication, attaching to the target database (opening relation and catalog caches lazily as
queries need them). Each backend keeps private memory for catalog caches, plan caches and
`work_mem` usage, and has a slot in shared arrays (`PGPROC`) that operations like snapshot creation
must consider, so the cost of some operations grows with the number of connections — much improved in
recent releases, but not zero.

PgBouncer is a single-threaded, event-driven proxy that speaks the PostgreSQL wire protocol on both
sides. It authenticates clients itself, keeps a pool of authenticated server connections per
`(database, user)` pair, and, in transaction mode, watches the protocol's `ReadyForQuery` messages:
when the server reports the transaction status as idle, it releases that server connection back to
the pool, optionally running `server_reset_query` (`DISCARD ALL`) — which only runs in session mode by
default. For prepared statements it rewrites statement names and re-prepares them on whichever server
connection a client lands on.

## Common mistakes

- Raising `max_connections` to thousands instead of pooling.
- Transaction pooling with session `SET`, temp tables, session advisory locks or `LISTEN`.
- Pool sizes chosen by client count rather than database capacity.
- Stacking large application pools behind PgBouncer so that PgBouncer's own limits are hit.
- Forgetting that `pg_stat_activity` behind a pooler shows the pooler's connections, not your clients —
  set `application_name` per transaction if you need attribution.

## Exercise

1. Install PgBouncer, point it at your lab database, and reproduce the `-C` comparison and the
   200-client test. Watch `SHOW POOLS` during the 200-client run.
2. Find the number of direct client connections at which throughput peaks on your machine for
   `pgbench -S`, by running 4, 8, 16, 32, 64 clients.
3. Reproduce the session-state leak with `search_path` instead of `statement_timeout`, then show that
   `SET LOCAL` fixes it.
4. Configure PgBouncer with `auth_type = scram-sha-256` and an `auth_query` against the server instead of
   a `trust` lab setup.
