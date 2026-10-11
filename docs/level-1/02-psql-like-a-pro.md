---
description: "Use psql efficiently: describe commands, expanded output, \\gset and \\gexec, \\copy, \\watch, \\crosstabview, ON_ERROR_STOP scripts and a sensible .psqlrc."
---

# 02 · psql Like a Pro

GUI tools come and go, but `psql` is on every server you will ever SSH into, it is what the
documentation's examples assume, and it can do things most GUIs cannot: loop over query results,
turn a query into more queries, and run safely inside shell scripts and CI. Twenty minutes learning
it properly pays for itself the first time production is on fire and the only thing you have is a
terminal.

Every output in this lesson was captured from `psql` 18.6 against the `shop` database from
lesson 1.

## Connecting

```bash
psql -h localhost -p 5432 -U postgres -d shop
psql "postgresql://postgres@localhost:5432/shop?sslmode=disable"   # URI form
```

Connection settings can also come from environment variables (`PGHOST`, `PGPORT`, `PGUSER`,
`PGDATABASE`, `PGPASSWORD`), from a service file (`~/.pg_service.conf`, used with
`psql service=reporting`) and passwords from `~/.pgpass` (one line per server:
`host:port:database:user:password`, file mode `0600` or psql ignores it). Prefer `.pgpass` over
`PGPASSWORD`, which is visible to other users through process environments on some systems.

The same variables are honoured by every libpq-based tool — `pg_dump`, `pgbench`, many drivers —
so setting them once covers the whole toolchain.

## Two kinds of input

Anything ending in `;` is SQL and goes to the server. Anything starting with a backslash is a
**meta-command** that psql handles itself. `\?` lists the meta-commands; `\h CREATE INDEX` prints
SQL syntax help without leaving the terminal.

## Describing things: the `\d` family

```text
shop=# \d customers
                       Table "public.customers"
 Column |  Type  | Collation | Nullable |           Default
--------+--------+-----------+----------+------------------------------
 id     | bigint |           | not null | generated always as identity
 email  | text   |           | not null |
Indexes:
    "customers_pkey" PRIMARY KEY, btree (id)
```

| Command | Lists |
|---------|-------|
| `\l` | databases |
| `\dn` | schemas |
| `\dt`, `\dv`, `\dm`, `\ds` | tables, views, materialized views, sequences |
| `\di` | indexes |
| `\df` | functions |
| `\du` | roles |
| `\dx` | installed extensions |
| `\dp` | access privileges |
| `\d+ name` | an object with extra detail: storage, size, description |

Most take a pattern: `\dt sales*`, `\df *json*`, `\dt billing.*`. Add `S` to include system
objects (`\dfS now`).

A habit worth forming: run `\set ECHO_HIDDEN on` once and psql prints the catalog query behind each
`\d` command. It is the fastest way to learn the system catalogs.

## Output formatting

```text
shop=# \x auto
Expanded display is used automatically.
```

`\x auto` switches to one-column-per-line display only when rows are too wide for your terminal —
the single most useful setting for reading `pg_stat_activity`. Also useful:

```text
shop=# \pset null '∅'
Null display is "∅".
shop=# SELECT NULL AS nothing, 1 AS one;
 nothing | one
---------+-----
 ∅       |   1
(1 row)
```

By default NULL and the empty string look identical; making NULL visible removes a whole category
of debugging confusion.

`\crosstabview` pivots a result in the client — no `crosstab()` extension needed:

```text
shop=# SELECT region, quarter, amount FROM sales \crosstabview
 region | Q1  | Q2
--------+-----+-----
 north  | 120 | 150
 south  |  90 | 110
(2 rows)
```

Others: `\timing on` prints client-measured duration after each statement; `\pset format csv`
(or `--csv` on the command line) for machine-readable output; `\o file.txt` sends output to a
file until the next `\o`.

## Variables, `\gset` and `\gexec`

psql has its own variables, set with `\set` and interpolated with `:name`. `\gset` stores the
columns of a one-row result into variables named after the columns:

```text
shop=# SELECT count(*) AS n, max(id) AS top FROM customers \gset
shop=# \echo :n customers, highest id :top
1000 customers, highest id 1000
```

Interpolation has three forms: `:name` raw, `:'name'` as a quoted literal, `:"name"` as a quoted
identifier. Use the quoted forms whenever the value could contain anything unexpected.

`\gexec` runs a query, then executes **each cell of the result as a SQL statement**. It is how you
generate DDL from the catalogs:

```text
shop=# SELECT format('SELECT %L AS tbl, count(*) FROM %I', relname, relname)
shop-#   FROM pg_class WHERE relname IN ('customers') \gexec
    tbl    | count
-----------+-------
 customers |  1000
(1 row)
```

`format()` with `%I` (identifier) and `%L` (literal) quotes values correctly, so a table called
`Order Items` does not break the generated SQL. Real-world uses: `GRANT` on every table in a schema,
`VACUUM ANALYZE` a list of tables, or `REINDEX` every index matching a pattern.

`\bind` sends a statement with real parameters through the extended protocol — handy for
reproducing exactly what an application driver does:

```text
shop=# SELECT $1::int + $2::int AS total \bind 2 3 \g
 total
-------
     5
(1 row)
```

## `\watch`: re-run a query on a timer

```text
shop=# SELECT count(*) AS active FROM pg_stat_activity WHERE state = 'active' \watch i=1 c=2
Sun Oct 11 11:56:22 2026 (every 1s)

 active
--------
      1
(1 row)

Sun Oct 11 11:56:23 2026 (every 1s)

 active
--------
      1
(1 row)
```

`i=` sets the interval and `c=` the number of runs (omit it to run until Ctrl-C). Perfect for
watching a migration's progress or a replication lag figure.

## `\copy` versus `COPY`

```text
shop=# \copy (SELECT id, email FROM customers ORDER BY id LIMIT 3) TO 'cust.csv' WITH (FORMAT csv, HEADER)
COPY 3
shop=# \! cat cust.csv
id,email
1,user1@example.com
2,user2@example.com
3,user3@example.com
```

Server-side `COPY ... TO '/path'` makes the **server process** write a file on the **server's**
filesystem and needs superuser or the `pg_write_server_files` role. `\copy` runs the same `COPY`
but streams the data through your connection, so the file lands on *your* machine with *your*
permissions. For loading a local CSV into a remote database, `\copy` is almost always what you want.
`COPY` is also dramatically faster than many single-row `INSERT`s; Level 1 · 08 shows why.

## Editing and history

- `\e` opens the last query in `$EDITOR`; save and quit to run it. `\ef function_name` edits a
  function's source.
- `\s` shows command history; Ctrl-R searches it.
- `\i file.sql` runs a file; `\ir` runs it relative to the current script's directory.

## Errors: reading them and stopping on them

This was captured from a script file, so psql prefixes each message with the file name and line:

```text
SELECT * FROM nonexistent;
psql:b.sql:14: ERROR:  relation "nonexistent" does not exist
LINE 1: SELECT * FROM nonexistent;
                      ^
\errverbose
psql:b.sql:15: error: ERROR:  42P01: relation "nonexistent" does not exist
LINE 1: SELECT * FROM nonexistent;
                      ^
LOCATION:  parserOpenTable, parse_relation.c:1501
```

`\errverbose` re-prints the last error with its **SQLSTATE** code (`42P01`, undefined table) — the
stable identifier your application code should match on, rather than the English message.

By default, a script keeps going after an error. Here is a two-statement file whose first statement
fails:

```sql
-- d.sql
INSERT INTO sales (amount) VALUES ('lots');
SELECT 'still running' AS status;
```

```text
$ psql -X -d shop -f d.sql; echo "exit=$?"
psql:d.sql:1: ERROR:  invalid input syntax for type integer: "lots"
LINE 1: INSERT INTO sales (amount) VALUES ('lots');
                                           ^
    status
---------------
 still running
(1 row)

exit=0
```

The script "succeeded" with exit code 0. In a deploy pipeline that is a disaster. With
`ON_ERROR_STOP`:

```text
$ psql -X -d shop -v ON_ERROR_STOP=1 -f d.sql; echo "exit=$?"
psql:d.sql:1: ERROR:  invalid input syntax for type integer: "lots"
LINE 1: INSERT INTO sales (amount) VALUES ('lots');
                                           ^
exit=3
```

Exit code 3 means "an error occurred in a script". Add `--single-transaction` (`-1`) as well, and
the whole file runs inside one transaction, so a failure leaves nothing half-applied.

## A sensible `~/.psqlrc`

```text
\set QUIET 1
\pset null '∅'
\x auto
\set HISTSIZE 10000
\set HISTFILE ~/.psql_history- :DBNAME
\set COMP_KEYWORD_CASE upper
\set PROMPT1 '%n@%M:%>/%/ %x%# '
\set ON_ERROR_ROLLBACK interactive
\timing on
\unset QUIET
```

- `HISTFILE` with `:DBNAME` keeps a separate history per database.
- `PROMPT1`: `%n` user, `%M` host, `%>` port, `%/` database, `%x` shows `*` while you are inside a
  transaction — a constant reminder that you have something uncommitted.
- `ON_ERROR_ROLLBACK interactive` silently wraps each statement in a savepoint *when typing at the
  prompt*, so a typo inside a long `BEGIN` block does not abort the whole transaction. It stays off
  for scripts.

Scripts and CI jobs should use `psql -X`, which skips `.psqlrc`, so your personal settings cannot
change a script's behaviour.

## How It Actually Works

psql is an ordinary libpq client. For SQL it uses the **simple query protocol**: it sends the text,
and the server parses, plans and runs it, returning rows plus a "command tag" such as
`INSERT 0 1000` (that `0` is a historical OID field, always 0 now). Multiple statements in one
string are run in sequence. `\bind` switches to the **extended protocol** — Parse, Bind, Execute
messages — which is what drivers use for parameterised queries, and is why the `$1` placeholder
syntax works there and not in a plain query.

Meta-commands never reach the server as text. `\d customers` is translated into ordinary catalog
queries against `pg_class`, `pg_attribute` and `pg_index` (which `ECHO_HIDDEN` reveals). `\gset`
and `\gexec` are client-side loops over a result set. `\copy` sends `COPY ... FROM STDIN` or
`TO STDOUT` and pumps the file through the connection using the COPY sub-protocol.

Because psql waits for the whole result before printing by default, a `SELECT` returning millions
of rows can exhaust client memory. Setting `\set FETCH_COUNT 1000` makes psql use a cursor-like
mode and print in batches.

## Common mistakes

- **Forgetting the semicolon.** The prompt changes from `=#` to `-#`, meaning psql is waiting for
  more input. Type `;` or `\r` to reset the buffer.
- **Using `COPY ... FROM '/local/file'` against a remote server.** The path is resolved on the
  server. Use `\copy`.
- **Scripts without `ON_ERROR_STOP`**, as shown above — they "succeed" no matter what.
- **Building SQL with raw `:var` interpolation** of untrusted values. Use `:'var'` or `:"var"`.
- **Relying on `.psqlrc` in automation.** Use `-X` so scripts behave identically everywhere.

## Exercise

1. Write a `~/.psqlrc` with at least the null marker, `\x auto` and a prompt that shows the database
   and transaction status. Start a transaction and confirm the prompt changes.
2. Use `\gexec` to generate and run `SELECT count(*)` for every table in the `public` schema of your
   lab database, reading table names from `pg_tables`.
3. Export the 10 newest rows of a table to CSV with `\copy`, then load them into a new empty table
   with `\copy ... FROM`.
4. Write a two-statement script whose first statement fails. Run it with and without
   `-v ON_ERROR_STOP=1 -1` and record the exit codes and what was left in the database each time.
5. Turn on `ECHO_HIDDEN`, run `\di`, and explain each catalog table used in the hidden query.
