---
description: "What a PostgreSQL cluster really is — the postmaster, backend processes, data directory and system catalogs — and how to install and run one locally."
---

# 01 · How PostgreSQL Is Built: Clusters, Processes & Setup

Most SQL courses treat the database as a black box you send queries to. This course does the
opposite. PostgreSQL is unusually transparent: almost everything it does is visible as an ordinary
operating-system process, an ordinary file on disk, or a row in a catalog table you can query. Once
you can see those pieces, things that look like magic later — VACUUM, replication, crash recovery —
turn into mechanisms you can reason about.

This lesson gets a server running on your machine and then takes it apart.

!!! info "Where this course starts"
    You should already be comfortable with everyday SQL: `SELECT`, joins, `GROUP BY`, creating
    tables. If not, work through the
    [SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/) first — this course builds
    on it rather than repeating it.

## Vocabulary: cluster, database, schema

PostgreSQL uses three nested containers, and the words are easy to mix up:

| Term | What it is | Created by |
|------|------------|------------|
| **Cluster** | One running server: one data directory, one port, one set of roles, many databases | `initdb` |
| **Database** | An isolated namespace inside the cluster. A connection belongs to exactly one database | `CREATE DATABASE` |
| **Schema** | A folder for tables, functions and types *inside* one database | `CREATE SCHEMA` |

"Cluster" here has nothing to do with multiple machines — it is an old term for "a collection of
databases managed by one server". Roles (users) are cluster-wide; tables are per-database. You
cannot join a table in database `shop` to a table in database `billing` with plain SQL — that is
one of the reasons most applications use one database with several schemas (lesson 4).

## Installing PostgreSQL

Pick whichever route suits your machine. Every command in this course was run against
**PostgreSQL 18.6**; anything version-specific is called out.

=== "macOS (Homebrew)"

    ```bash
    brew install postgresql@18
    # keg-only: the binaries live here and are not on your PATH automatically
    export PATH="/opt/homebrew/opt/postgresql@18/bin:$PATH"
    ```

=== "Debian / Ubuntu"

    ```bash
    # the PGDG repository carries every supported major version
    sudo apt install -y postgresql-common
    sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
    sudo apt install -y postgresql-18
    ```

    Debian's packaging creates and starts a cluster for you and adds wrapper tools such as
    `pg_lsclusters` and `pg_ctlcluster`.

=== "Docker"

    ```bash
    docker run --name pg18 -e POSTGRES_PASSWORD=devpass -p 5432:5432 -d postgres:18
    docker exec -it pg18 psql -U postgres
    ```

Package managers usually create a cluster for you. To *see* how it works, create one by hand — it
takes two commands and you can delete the directory afterwards.

## Creating a cluster yourself

```bash
export LC_ALL=en_US.UTF-8          # see the pitfall below
initdb -D ~/pgdata -U postgres --auth=trust -E UTF8 --locale=en_US.UTF-8
pg_ctl -D ~/pgdata -l ~/pgdata.log start
psql -h localhost -U postgres -c "select version();"
```

`initdb` creates the data directory, the template databases and the `postgres` superuser.
`--auth=trust` means "no password for local connections" — acceptable on a laptop you alone use,
never on a server (lesson 5 replaces it).

When I first started the server for this course on macOS, it refused to start, and the log said:

```text
FATAL:  postmaster became multithreaded during startup
HINT:  Set the LC_ALL environment variable to a valid locale.
```

Exporting `LC_ALL=en_US.UTF-8` before `pg_ctl` fixed it. Two other real errors you may meet:

- `Unix-domain socket path "..." is too long (maximum 103 bytes)` — your data directory or
  `unix_socket_directories` path is too deep. Use a shorter path, or connect over TCP with
  `-h localhost`.
- `could not bind IPv4 address "127.0.0.1": Address already in use` — another server already owns
  port 5432. Set `port = 5433` in `postgresql.conf` or stop the other server.

## The process model: one process per connection

List the server's processes right after start-up:

```bash
ps -o pid,ppid,command -U $USER | grep "[p]ostgres"
```

```text
44133     1 /opt/homebrew/Cellar/postgresql@18/18.6_1/bin/postgres -D .../pgdata
44134 44133 postgres: io worker 0
44135 44133 postgres: io worker 1
44136 44133 postgres: io worker 2
44137 44133 postgres: checkpointer
44138 44133 postgres: background writer
44140 44133 postgres: walwriter
44141 44133 postgres: autovacuum launcher
44142 44133 postgres: logical replication launcher
```

(The data directory path is shortened here.) The first process is the **postmaster**. Every other
process is its child:

| Process | Job | Covered in |
|---------|-----|-----------|
| checkpointer | Periodically flushes dirty pages to data files and marks a safe restart point | Level 2 · 09 |
| background writer | Trickles dirty pages to disk so backends rarely have to | Level 2 · 09 |
| walwriter | Flushes the write-ahead log | Level 2 · 09 |
| autovacuum launcher | Starts autovacuum workers when tables need cleaning | Level 2 · 02 |
| logical replication launcher | Starts subscription workers | Level 4 · 04 |
| io worker | Performs asynchronous reads (new in PostgreSQL 18, `io_method = worker`) | Level 4 · 01 |

Now connect with `psql` and ask the server the same question from the inside:

```sql
SELECT pid, backend_type FROM pg_stat_activity ORDER BY pid;
```

```text
  pid  |         backend_type
-------+------------------------------
 44134 | io worker
 44135 | io worker
 44136 | io worker
 44137 | checkpointer
 44138 | background writer
 44140 | walwriter
 44141 | autovacuum launcher
 44142 | logical replication launcher
 44439 | client backend
(9 rows)
```

The new row, `client backend`, is *your* connection. When a client connects, the postmaster forks a
dedicated backend process for it, and that process does all the parsing, planning and executing for
that one session until it disconnects. `SELECT pg_backend_pid();` tells you which one is yours.

This design has consequences you will meet throughout the course:

- A crash in one backend cannot corrupt another backend's memory — but the postmaster still
  restarts every backend to be safe, because they share memory.
- Each connection costs a process plus its private memory, so thousands of idle connections are
  expensive. That is why connection poolers exist (Level 4 · 02).
- Backends communicate through **shared memory**: the buffer cache (`shared_buffers`), the lock
  table and the WAL buffers.

## The data directory on disk

```text
PG_VERSION   base/        global/      pg_hba.conf   pg_ident.conf  pg_wal/   pg_xact/
pg_multixact/ pg_logical/ pg_replslot/ pg_stat/      pg_tblspc/     postgresql.conf
postgresql.auto.conf  postmaster.opts  postmaster.pid  ...
```

The parts worth knowing now:

- `base/` — one sub-directory per database, named by the database's OID.
- `global/` — cluster-wide catalogs such as `pg_database` and `pg_authid` (roles).
- `pg_wal/` — the write-ahead log, 16 MB segment files. **Never delete files here by hand**; a
  "disk full, let me clean up pg_wal" moment is a classic way to destroy a database.
- `pg_xact/` — commit status for every transaction (did transaction 1234 commit or abort?).
- `postgresql.conf`, `postgresql.auto.conf`, `pg_hba.conf` — configuration (lesson 5 and Level 4).

Watch an object go from SQL to a file:

```sql
CREATE DATABASE shop;
SELECT oid, datname FROM pg_database ORDER BY oid;
```

```text
  oid  |  datname
-------+-----------
     1 | template1
     4 | template0
     5 | postgres
 16384 | shop
(4 rows)
```

```sql
\c shop
CREATE TABLE customers (
  id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email text NOT NULL
);
INSERT INTO customers (email)
SELECT 'user' || g || '@example.com' FROM generate_series(1, 1000) g;

SELECT pg_relation_filepath('customers');
```

```text
 pg_relation_filepath
----------------------
 base/16384/16386
(1 row)
```

```bash
ls -l ~/pgdata/base/16384/16386*
```

```text
-rw-------  1 you  staff  65536 Oct 11 11:54 .../base/16384/16386
-rw-------  1 you  staff  24576 Oct 11 11:54 .../base/16384/16386_fsm
```

(Owner, group and path are shortened; the sizes are as observed.)

The table *is* that 64 kB file — eight 8 kB pages. The `_fsm` file is its free-space map. The
primary key index is a separate file again, which is why the two size functions disagree:

```sql
SELECT pg_size_pretty(pg_relation_size('customers'))       AS heap,
       pg_size_pretty(pg_total_relation_size('customers')) AS total;
```

```text
 heap  | total
-------+--------
 64 kB | 136 kB
(1 row)
```

## Template databases

`CREATE DATABASE shop` does not build a database from nothing: it copies `template1`. Anything you
put in `template1` (an extension, a table) appears in every database created afterwards.
`template0` is a pristine copy that you should never modify; use it when you need a clean database
with a different encoding or locale, or when restoring a dump:

```sql
CREATE DATABASE restore_target TEMPLATE template0;
```

Copying a template requires that nobody is connected to it. If someone has a session open on
`template1`, `CREATE DATABASE` fails with "source database "template1" is being accessed by other
users".

## Starting, stopping and reloading

```bash
pg_ctl -D ~/pgdata status
pg_ctl -D ~/pgdata reload           # re-read config files without dropping connections
pg_ctl -D ~/pgdata stop -m fast     # default mode: roll back sessions, checkpoint, exit
pg_ctl -D ~/pgdata restart
```

`stop` has three modes. **smart** waits for every client to disconnect (it can wait forever).
**fast** — the default — rolls back open transactions, disconnects clients, writes a checkpoint and
exits cleanly. **immediate** kills everything without a checkpoint; the next start performs crash
recovery from the WAL. Use immediate only when fast hangs.

From SQL, `SELECT pg_reload_conf();` does the same as `pg_ctl reload`.

## How It Actually Works

When `pg_ctl start` runs, it launches `postgres -D <dir>`. The postmaster reads the configuration,
allocates the shared-memory segment (buffer cache, WAL buffers, lock tables), writes its PID to
`postmaster.pid` so a second server cannot start on the same directory, and checks `pg_control` —
an 8 kB file recording the last checkpoint location and whether the previous shutdown was clean.
If it was not clean, the startup process replays WAL from the last checkpoint before accepting
connections. Then it forks the auxiliary processes and starts listening.

A connection arrives: the postmaster accepts the socket and forks a child. The child — not the
postmaster — performs authentication against `pg_hba.conf`, loads the target database's catalog
caches and then loops: read a query, parse it, rewrite it (views and rules), plan it, execute it,
send rows back. The postmaster stays deliberately simple, so that a bug in query execution cannot
take down the process that supervises everything.

Data files are read and written in 8 kB **pages**. A backend that needs a page first looks in
`shared_buffers`; on a miss it reads the page from the operating system (which has its own page
cache) into a free buffer. Changes are made in the buffer and recorded in the WAL; the data file
itself is written later by the background writer or checkpointer. That split — log first, data
later — is what lets PostgreSQL survive a crash, and Level 2 · 09 follows it in detail.

## Common mistakes

- **Confusing database and schema.** Splitting one application across several *databases* makes
  cross-joins impossible; separate schemas in one database is usually what you wanted.
- **Running as the operating-system root user.** `initdb` and `postgres` refuse to run as root on
  purpose; create or use a dedicated `postgres` OS user on servers.
- **Deleting WAL files to free disk space.** WAL that has not been checkpointed is required for
  recovery. Find out *why* it is accumulating (often an abandoned replication slot — Level 4 · 03).
- **Leaving `trust` authentication in place** on anything reachable from a network.
- **Two servers, one port.** `psql` connects you to whichever server owns the port, which may not
  be the one you just configured. `SELECT version(), current_setting('data_directory');` settles it.

## Exercise

1. Create a throwaway cluster in a new directory on a non-default port (for example 5440).
2. Connect, create a database named `lab`, and record its OID.
3. Create a table, insert 10,000 rows, and find its file with `pg_relation_filepath`. Check the file
   size with `ls -l` and compare it with `pg_relation_size`.
4. Open a second `psql` session and find both backends in `pg_stat_activity`. Identify which PID is
   which with `pg_backend_pid()`.
5. Stop the server with `-m immediate`, start it again, and read the log. Find the lines showing that
   the server noticed the unclean shutdown and ran recovery.
