---
description: "Level 2 project: replay a help-desk workload with pgbench, find the worst queries with pg_stat_statements, fix them and measure the gain."
---

# 10 · Project — Diagnose a Slow Database

Here is the situation you will eventually be dropped into: an application that "has got slow",
nobody knows why, and you have database access. This project simulates that end to end. You will
build a help-desk database with realistic volumes and one inherited, unhelpful index, replay a
mixed application workload against it with `pgbench`, find the expensive statements with
`pg_stat_statements`, read their plans, fix them, and prove the improvement with numbers.

Everything below was run on PostgreSQL 18.6 on a laptop; the absolute numbers are specific to that
machine, but the method and the ratios are what you should expect anywhere.

## Step 0 — enable pg_stat_statements

It must be preloaded, so it needs a restart:

```text
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
```

```sql
CREATE EXTENSION pg_stat_statements;   -- in the database you want to query it from
```

It records every normalised statement (`WHERE id = 42` and `WHERE id = 7` become `WHERE id = $1`),
with call counts, total and mean time, rows, and buffer usage. It is the first thing to enable on any
server you are responsible for.

## Step 1 — the database

```sql
CREATE TABLE accounts (id int PRIMARY KEY, name text NOT NULL, plan text NOT NULL);
CREATE TABLE users (id int PRIMARY KEY, account_id int NOT NULL REFERENCES accounts, email text NOT NULL);
CREATE TABLE tickets (
  id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  account_id   int NOT NULL REFERENCES accounts,
  requester_id int NOT NULL REFERENCES users,
  status       text NOT NULL,
  priority     int NOT NULL,
  subject      text NOT NULL,
  created_at   timestamptz NOT NULL,
  updated_at   timestamptz NOT NULL
);
CREATE TABLE comments (
  id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  ticket_id  bigint NOT NULL REFERENCES tickets,
  author_id  int NOT NULL REFERENCES users,
  body       text NOT NULL,
  created_at timestamptz NOT NULL
);
CREATE INDEX tickets_status_idx ON tickets (status);   -- added "for the dashboard" years ago

SELECT setseed(0.7);
INSERT INTO accounts SELECT g, 'Account ' || g, (ARRAY['free','team','enterprise'])[1 + g % 3]
FROM generate_series(1, 20000) g;
INSERT INTO users SELECT g, 1 + (g % 20000), 'Agent' || g || '@Desk.example'
FROM generate_series(1, 200000) g;
INSERT INTO tickets (account_id, requester_id, status, priority, subject, created_at, updated_at)
SELECT a, 1 + ((a - 1) + 20000 * (g % 10)) % 200000,
       CASE WHEN random() < 0.85 THEN 'closed' WHEN random() < 0.6 THEN 'open' ELSE 'pending' END,
       1 + (random() * 3)::int, 'Ticket ' || g, ts, ts + random() * interval '10 days'
FROM (SELECT g, 1 + (random() * 19999)::int AS a,
             timestamptz '2024-01-01' + random() * interval '1000 days' AS ts
      FROM generate_series(1, 2000000) g) s;
INSERT INTO comments (ticket_id, author_id, body, created_at)
SELECT 1 + (random() * 1999999)::bigint, 1 + (random() * 199999)::int,
       repeat('comment text ', 8), timestamptz '2024-01-01' + random() * interval '1000 days'
FROM generate_series(1, 4000000);
VACUUM ANALYZE;
```

Loading took about a minute and a half. Resulting sizes:

```text
 relname  | pg_size_pretty
----------+----------------
 accounts | 1576 kB
 users    | 17 MB
 tickets  | 233 MB
 comments | 724 MB
```

Ticket status mix: 1.7 million `closed`, 180,000 `open`, 120,000 `pending`.

## Step 2 — the workload

Five pgbench script files, each modelling an application screen. pgbench's `\set` draws random
parameters, so every execution hits different rows.

```sql
-- inbox.sql: an agent's open tickets
\set acct random(1, 20000)
SELECT id, subject, status, updated_at FROM tickets
WHERE account_id = :acct AND status IN ('open', 'pending')
ORDER BY updated_at DESC LIMIT 20;
```

```sql
-- ticket_page.sql: the conversation on one ticket
\set t random(1, 2000000)
SELECT c.id, c.body, c.created_at, u.email FROM comments c JOIN users u ON u.id = c.author_id
WHERE c.ticket_id = :t ORDER BY c.created_at;
```

```sql
-- login.sql: case-insensitive email lookup
\set u random(1, 200000)
SELECT id, account_id FROM users WHERE lower(email) = lower('agent' || :u || '@desk.example');
```

```sql
-- dashboard.sql: last 90 days by status
\set acct random(1, 20000)
SELECT status, count(*) FROM tickets
WHERE account_id = :acct AND created_at >= timestamptz '2026-09-27' - interval '90 days'
GROUP BY status;
```

```sql
-- reply.sql: an agent replies
\set t random(1, 2000000)
\set u random(1, 200000)
BEGIN;
INSERT INTO comments (ticket_id, author_id, body, created_at) VALUES (:t, :u, 'reply', now());
UPDATE tickets SET updated_at = now(), status = 'pending' WHERE id = :t;
COMMIT;
```

Run them mixed 30/30/20/10/10, with 8 concurrent clients for 30 seconds:

```bash
pgbench -n -c 8 -j 4 -T 30 \
  -f inbox.sql@30 -f ticket_page.sql@30 -f login.sql@20 -f dashboard.sql@10 -f reply.sql@10 desk
```

## Step 3 — the baseline

```text
number of transactions actually processed: 888
latency average = 274.820 ms
tps = 29.109955 (without initial connection time)
SQL script 1: inbox.sql        - latency average = 166.228 ms
SQL script 2: ticket_page.sql  - latency average = 646.800 ms
SQL script 3: login.sql        - latency average = 64.389 ms
SQL script 4: dashboard.sql    - latency average = 225.135 ms
SQL script 5: reply.sql        - latency average = 4.406 ms
```

29 transactions per second, and opening a ticket takes two-thirds of a second. Now ask the database
where the time went. (My first attempt at this query failed with
`function round(double precision, integer) does not exist` — the time columns are `double precision`,
and the two-argument `round` exists only for `numeric`, hence the casts.)

```sql
SELECT calls, round(total_exec_time) AS total_ms, round(mean_exec_time::numeric, 2) AS mean_ms,
       round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 1) AS pct,
       (shared_blks_hit + shared_blks_read) / calls AS blks_per_call,
       left(regexp_replace(query, '\s+', ' ', 'g'), 60) AS query
FROM pg_stat_statements
WHERE dbid = (SELECT oid FROM pg_database WHERE datname = 'desk')
ORDER BY total_exec_time DESC LIMIT 6;
```

```text
 calls | total_ms | mean_ms | pct  | blks_per_call |                            query
-------+----------+---------+------+---------------+--------------------------------------------------------------
   258 |   166462 |  645.20 | 69.0 |         81470 | SELECT c.id, c.body, c.created_at, u.email FROM comments c J
   262 |    43182 |  164.82 | 17.9 |         22960 | SELECT id, subject, status, updated_at FROM tickets WHERE ac
    91 |    20315 |  223.24 |  8.4 |         22758 | SELECT status, count(*) FROM tickets WHERE account_id = $1 A
   178 |    11118 |   62.46 |  4.6 |          1568 | SELECT id, account_id FROM users WHERE lower(email) = lower(
    99 |      118 |    1.20 |  0.0 |            24 | INSERT INTO comments (ticket_id, author_id, body, created_at
    99 |       13 |    0.13 |  0.0 |            15 | UPDATE tickets SET updated_at = now(), status = $1 WHERE id
```

Sort by **total** time, not mean: a 1 ms query called a million times matters more than a 2-second
query called once a day. Here the ticket page is 69% of all database time, and `blks_per_call` —
81,470 pages, about 640 MB touched to show one ticket — says why before you even open a plan.

## Step 4 — read the plans

**Ticket page:**

```text
 Gather Merge (actual time=480.015..481.462 rows=0.00 loops=1)
   Buffers: shared hit=4574 read=77075
   ->  Sort ...
         ->  Nested Loop (actual time=474.381..474.383 rows=0.00 loops=3)
               ->  Parallel Seq Scan on comments c (actual time=474.370..474.370 rows=0.00 loops=3)
                     Filter: (ticket_id = 123456)
                     Rows Removed by Filter: 1333366
```

A full scan of 4 million comments per page view. The cause is the most common missing index in
PostgreSQL: **foreign-key columns are not indexed automatically** (Level 1 · 06).

**Inbox:**

```text
 Limit (actual time=281.176..282.759 rows=17.00 loops=1)
   ->  Parallel Bitmap Heap Scan on tickets (actual time=70.814..279.326 rows=5.67 loops=3)
         Recheck Cond: (status = ANY ('{open,pending}'::text[]))
         Filter: (account_id = 777)
         Rows Removed by Filter: 100039
         ->  Bitmap Index Scan on tickets_status_idx (actual time=31.636..31.636 rows=300149.00 loops=1)
```

The old status index matches 300,149 tickets for *every* account, then discards all but 17. The
selective column, `account_id`, has no index.

**Dashboard:** a parallel sequential scan of all 2 million tickets, removing 666,661 rows per process.

**Login:** `lower(email)` with no expression index — a sequential scan of 200,000 users (lesson 4).

## Step 5 — fix, without locking the application out

On a live system, use `CREATE INDEX CONCURRENTLY`, which does not block writes (it takes longer and
cannot run inside a transaction block):

```sql
SET maintenance_work_mem = '512MB';
CREATE INDEX CONCURRENTLY comments_ticket_created_idx ON comments (ticket_id, created_at);
CREATE INDEX CONCURRENTLY tickets_account_open_idx ON tickets (account_id, updated_at DESC)
  WHERE status IN ('open', 'pending');
CREATE INDEX CONCURRENTLY tickets_account_created_idx ON tickets (account_id, created_at) INCLUDE (status);
CREATE UNIQUE INDEX CONCURRENTLY users_email_lower_idx ON users (lower(email));
DROP INDEX CONCURRENTLY tickets_status_idx;
ANALYZE users;
```

```text
CREATE INDEX  Time: 3409.880 ms (00:03.410)
CREATE INDEX  Time: 454.521 ms
CREATE INDEX  Time: 1478.610 ms (00:01.479)
CREATE INDEX  Time: 740.229 ms
DROP INDEX    Time: 4.328 ms
```

The reasoning behind each:

- `comments (ticket_id, created_at)` — equality then sort column, so the comments come out already
  ordered (lesson 4). It also makes deleting a ticket cheap.
- `tickets (account_id, updated_at DESC) WHERE status IN ('open','pending')` — a *partial* index of
  only the 300,000 live tickets: 18 MB, against 77 MB for the full-table dashboard index below, and
  exactly the rows the inbox wants.
- `tickets (account_id, created_at) INCLUDE (status)` — lets the dashboard run as an index-only scan.
- The unique expression index on `lower(email)` both speeds up login and **enforces** that two users
  cannot differ only by case — a bug the data had been free to contain.
- `tickets_status_idx` was useless for every query (status has three values) and cost write work on
  every status change, so it goes.
- `ANALYZE users` collects statistics for the new expression.

## Step 6 — verify each plan

```text
-- ticket page
 Nested Loop (actual time=0.290..0.290 rows=0.00 loops=1)
   Buffers: shared hit=3 read=3
   ->  Index Scan using comments_ticket_created_idx on comments c
 Execution Time: 0.610 ms

-- inbox
 Limit (actual time=0.140..0.141 rows=17.00 loops=1)
   Buffers: shared hit=3 read=17
   ->  Sort ...
         ->  Bitmap Heap Scan on tickets
               ->  Bitmap Index Scan on tickets_account_open_idx
 Execution Time: 0.150 ms

-- dashboard
 HashAggregate (actual time=0.063..0.063 rows=2.00 loops=1)
   ->  Index Only Scan using tickets_account_created_idx on tickets
         Heap Fetches: 0
 Execution Time: 0.075 ms

-- login
 Index Scan using users_email_lower_idx on users (actual time=0.014..0.014 rows=1.00 loops=1)
 Execution Time: 0.017 ms
```

Every query now touches a handful of pages instead of tens of thousands.

## Step 7 — measure again, the same way

```text
number of transactions actually processed: 677611
latency average = 0.354 ms
tps = 22584.123745 (without initial connection time)
SQL script 1: inbox.sql        - latency average = 0.374 ms
SQL script 2: ticket_page.sql  - latency average = 0.375 ms
SQL script 3: login.sql        - latency average = 0.134 ms
SQL script 4: dashboard.sql    - latency average = 0.236 ms
SQL script 5: reply.sql        - latency average = 0.782 ms
```

| | Before | After |
|---|---|---|
| Throughput | 29 tps | 22,584 tps |
| Ticket page | 646.8 ms | 0.375 ms |
| Inbox | 166.2 ms | 0.374 ms |
| Dashboard | 225.1 ms | 0.236 ms |
| Login | 64.4 ms | 0.134 ms |
| Reply (write) | 4.4 ms | 0.78 ms |

Even the write got faster — not because writes are cheaper (they now maintain more indexes) but
because they no longer compete for CPU and I/O with full-table scans.

## Step 8 — check what the fix cost

```text
 relname | n_tup_upd | n_tup_hot_upd
---------+-----------+---------------
 tickets |     67556 |             0
```

No HOT updates (lesson 3). Every reply updates `updated_at` and `status`, and both are now part of
`tickets_account_open_idx` (as key and predicate), so every update must insert new index entries.
(Before the fix, `status` was indexed too, so it was not HOT then either.) Here that is the right
trade: reads outnumber writes 9 to 1 and the write still takes under a millisecond. On a write-heavy
table you would weigh it, perhaps with an index on `(account_id) WHERE status IN (...)` and a sort.

Also check that nothing you created is dead weight:

```text
        indexrelname         | idx_scan | pg_size_pretty
-----------------------------+----------+----------------
 comments_ticket_created_idx |   202827 | 120 MB
 tickets_account_created_idx |    67889 | 77 MB
 tickets_account_open_idx    |   204003 | 18 MB
 users_email_lower_idx       |   135439 | 8848 kB
```

## How It Actually Works

`pg_stat_statements` hooks into the executor. After parsing, it computes a **query ID** from the
normalised parse tree (constants removed), so statements that differ only in literal values share
one entry. At the end of execution it adds that run's timing, row count and buffer counters to the
entry in shared memory (persisted across clean restarts). Because it aggregates by query shape, it
shows *total cost to the system*, which is what you want for prioritising.

`pgbench` runs each client in a loop: pick a script by weight, substitute the `\set` variables, send
the statements, wait for results. Latency is measured on the client side, so it includes network
round trips — with 8 clients on a laptop that also runs the server, the client and server compete for
the same CPU, which is one reason not to treat these numbers as a server benchmark.

`CREATE INDEX CONCURRENTLY` builds the index in phases: it registers the index as not-yet-valid
(so new writes start maintaining it), waits for transactions that might not know about it to finish,
does one table scan to build it, waits again, does a second pass to add rows inserted during the
first, and finally marks it valid. If it fails part-way it leaves an `INVALID` index behind that you
must drop and retry — check `pg_index.indisvalid` after any concurrent build.

## Extensions

1. Write a `close_ticket.sql` script that closes 5% of open tickets and add it to the mix. What happens
   to the size of `tickets_account_open_idx` over a few runs, and why?
2. Replace `tickets_account_open_idx` with `(account_id) WHERE status IN ('open','pending')` and
   compare read latency, write latency and HOT ratio.
3. Run the workload with 32 clients. Where does throughput stop scaling, and what does
   `pg_stat_activity.wait_event` show during the run?
4. Turn on `auto_explain` with `auto_explain.log_min_duration = '50ms'` (Level 4 · 06), revert one fix,
   and find the slow plan in the server log.

## Exercise

Repeat the whole project on your own machine, recording your own before/after table. Then write a
one-page incident report as if this were production: symptoms, evidence (the `pg_stat_statements`
output and plans), root causes, fixes, verification, and follow-up actions to stop it recurring
(for example, a CI check that every foreign-key column has an index).
