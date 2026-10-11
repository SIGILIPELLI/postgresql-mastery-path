---
description: "Logical replication: publications with row filters and column lists, replica identity rules, DDL breakage, and change data capture."
---

# 04 · Logical Replication & Change Data Capture

Physical replication (lesson 3) copies a whole cluster block for block. **Logical replication** copies
*row changes* for chosen tables: "insert this row", "update the row with key 7". That makes it possible
to replicate a subset of tables, filter rows and columns, replicate between different major versions
(the basis of near-zero-downtime upgrades, lesson 9), consolidate many databases into one, and feed
changes to other systems — **change data capture (CDC)**.

It is also more fragile than physical replication, and building this lesson broke it in three
different ways. All on PostgreSQL 18.6, publisher on port 54329 (database `lab`), subscriber on
54330 (the standby promoted in lesson 3, database `analytics`).

## Requirements

The publisher needs `wal_level = logical` (a restart-only setting; the cluster used to write this course had it set from the start),
and free `max_replication_slots` and `max_wal_senders`. The connecting role needs the `REPLICATION`
attribute (or to be the table owner) **and** `SELECT` on the published tables.

## A publication with a row filter and a column list

Replicate only Indian customers, and not their email addresses or credit limits:

```sql
-- publisher, database lab
CREATE TABLE customers_lr (id int PRIMARY KEY, name text NOT NULL, email text,
                           country text NOT NULL, credit_limit numeric);
INSERT INTO customers_lr VALUES (1, 'Ana', 'ana@example.com', 'IN', 5000),
                                (2, 'Ben', 'ben@example.com', 'US', 3000),
                                (3, 'Chen', 'chen@example.com', 'IN', 7000);

CREATE PUBLICATION india_customers
  FOR TABLE customers_lr (id, name, country) WHERE (country = 'IN');
SELECT * FROM pg_publication_tables;
```

```text
     pubname     | schemaname |  tablename   |     attnames      |       rowfilter
-----------------+------------+--------------+-------------------+------------------------
 india_customers | public     | customers_lr | {id,name,country} | (country = 'IN'::text)
```

Row filters and column lists arrived in PostgreSQL 15. On the subscriber, the table must already exist
— **logical replication does not create or change tables** — with at least the published columns (it
may have extra ones):

```sql
-- subscriber, database analytics
CREATE TABLE customers_lr (id int PRIMARY KEY, name text NOT NULL, country text NOT NULL,
                           synced_at timestamptz DEFAULT now());
CREATE SUBSCRIPTION india_sub
  CONNECTION 'host=localhost port=54329 dbname=lab user=replicator password=repl-dev'
  PUBLICATION india_customers;
```

```text
NOTICE:  created replication slot "india_sub" on publisher
CREATE SUBSCRIPTION
```

## Breakage 1: permissions on the initial copy

The subscriber table stayed empty. Its log said why:

```text
ERROR:  could not start initial contents copy for table "public.customers_lr": ERROR:  permission denied for table customers_lr
```

The subscription first copies existing rows with a `COPY` run as the connecting role, which needs
`SELECT`. After `GRANT SELECT ON customers_lr TO replicator`, the sync worker retried on its own:

```text
 id | name | country
----+------+---------
  1 | Ana  | IN
  3 | Chen | IN
```

Only the Indian customers, and only three columns.

## Breakage 2: the publication broke the application's UPDATEs

Then I ran ordinary application statements on the **publisher**:

```text
UPDATE customers_lr SET name = 'Ana R.' WHERE id = 1;
ERROR:  cannot update table "customers_lr"
DETAIL:  Column used in the publication WHERE expression is not part of the replica identity.
DELETE FROM customers_lr WHERE id = 3;
ERROR:  cannot delete from table "customers_lr"
DETAIL:  Column used in the publication WHERE expression is not part of the replica identity.
```

Creating a publication made the production table reject updates and deletes. To replicate an UPDATE or
DELETE, the publisher must send enough of the old row — its **replica identity**, by default the
primary key — for the subscriber to find the row. To evaluate the row filter on the old row, the
filtered column (`country`) must be in that identity. It is not, so PostgreSQL refuses the write
rather than replicate it incorrectly.

The obvious fix, `REPLICA IDENTITY FULL` (send the whole old row), failed differently:

```text
ALTER TABLE customers_lr REPLICA IDENTITY FULL;
UPDATE customers_lr SET name = 'Ana R.' WHERE id = 1;
ERROR:  cannot update table "customers_lr"
DETAIL:  Column list used by the publication does not cover the replica identity.
```

With a column list, the replica identity must fit inside the published columns — and `FULL` means
*all* columns, including the email and credit limit we chose not to publish. The working fix is an
identity made of published, filter-covering columns:

```sql
CREATE UNIQUE INDEX customers_lr_id_country ON customers_lr (id, country);
ALTER TABLE customers_lr REPLICA IDENTITY USING INDEX customers_lr_id_country;
```

Lesson: **test every write path on the publisher after creating or changing a publication**, before
production does it for you.

## Changes flow — including rows crossing the filter

```sql
UPDATE customers_lr SET name = 'Ana R.' WHERE id = 1;
UPDATE customers_lr SET credit_limit = 9999 WHERE id = 3;   -- unpublished column
UPDATE customers_lr SET country = 'US' WHERE id = 4;        -- leaves the filter
UPDATE customers_lr SET country = 'IN' WHERE id = 5;        -- enters the filter
DELETE FROM customers_lr WHERE id = 3;
```

```text
 id |  name  | country | tier
----+--------+---------+------
  1 | Ana R. | IN      |
  5 | Erik   | IN      |
  6 | Farah  | IN      | gold
```

An update that moves a row *out* of the filter is sent as a delete (Divya, id 4, vanished); one that
moves a row *in* is sent as an insert (Erik, id 5, appeared). (Farah and the `tier` column come from
the next section.)

## Breakage 3: DDL is not replicated

On the publisher I added a column, included it in the publication, and inserted a row:

```sql
ALTER TABLE customers_lr ADD COLUMN tier text;
ALTER PUBLICATION india_customers SET TABLE customers_lr (id, name, country, tier) WHERE (country = 'IN');
INSERT INTO customers_lr VALUES (6, 'Farah', 'f@example.com', 'IN', 100, 'gold');
```

The subscriber's apply worker stopped, retrying and failing every few seconds:

```text
ERROR:  logical replication target relation "public.customers_lr" is missing replicated column: "tier"
ERROR:  logical replication target relation "public.customers_lr" is missing replicated column: "tier"
```

```text
  subname  | apply_error_count | sync_error_count
-----------+-------------------+------------------
 india_sub |                 2 |                3
```

While it is stuck, **nothing** replicates for that subscription, and the publisher retains WAL for its
slot (lesson 3's disk-filling risk). Adding the column on the subscriber let it resume and apply Farah's
row. The safe order for schema changes is: **add on the subscriber first, then the publisher**; for
drops, the reverse. Many teams run every migration on both sides from the same tool.

Other things that are not replicated: sequences' current values (sync them before a cutover),
`TRUNCATE` only if published, large objects, and materialized-view refreshes.

## Conflicts

The subscriber is an ordinary writable database. If someone inserts a row there that a replicated
insert later collides with, the apply worker errors on the unique violation and stops, exactly like
the DDL case. PostgreSQL 18 records conflict counts per type in `pg_stat_subscription_stats`; resolving
means fixing the data on the subscriber, or skipping the failing transaction
(`ALTER SUBSCRIPTION ... SKIP (lsn = ...)`, using the LSN from the error log). Treat subscriber tables as
read-only to applications unless you have designed for writes on both sides.

## Change data capture: reading the stream yourself

Subscriptions are one consumer of **logical decoding**. You can read the decoded change stream
directly through a slot and an output plugin. With the built-in `test_decoding` plugin:

```sql
SELECT slot_name FROM pg_create_logical_replication_slot('cdc_demo', 'test_decoding');

BEGIN;
INSERT INTO repl_demo VALUES (10, 'cdc insert');
UPDATE repl_demo SET note = 'cdc update' WHERE id = 10;
COMMIT;
BEGIN;
DELETE FROM repl_demo WHERE id = 10;
ROLLBACK;

SELECT lsn, xid, data FROM pg_logical_slot_get_changes('cdc_demo', NULL, NULL);
```

```text
    lsn     |  xid   |                                  data
------------+--------+------------------------------------------------------------------------
 2/54ADA1E0 | 467032 | BEGIN 467032
 2/54ADA1E0 | 467032 | table public.repl_demo: INSERT: id[integer]:10 note[text]:'cdc insert'
 2/54ADA2E8 | 467032 | table public.repl_demo: UPDATE: id[integer]:10 note[text]:'cdc update'
 2/54ADA370 | 467032 | COMMIT 467032
```

Only the committed transaction appears — the rolled-back delete never happened as far as consumers are
concerned — and changes arrive grouped by transaction in commit order. `get_changes` consumes them: a
second call returned 0 rows. (`pg_logical_slot_peek_changes` reads without consuming.)

Production CDC tools (Debezium is the best known; not run here) use the `pgoutput` plugin — the same one
subscriptions use — through the streaming protocol, and turn changes into messages for Kafka or similar
systems. The **outbox pattern** pairs well with it: write domain events into an `outbox` table in the
same transaction as the business change, and stream only that table.

Always drop slots you no longer consume — a forgotten logical slot retains WAL *and* holds back
`catalog_xmin`, which prevents vacuuming of system catalogs:

```sql
SELECT pg_drop_replication_slot('cdc_demo');
DROP SUBSCRIPTION india_sub;   -- also drops its slot on the publisher
```

## How It Actually Works

Logical decoding reads the same WAL that physical replication ships, but interprets it: a **walsender**
process with a logical slot decodes WAL records into row changes using catalog information *as of the
time of each change* (the slot's `catalog_xmin` keeps old catalog row versions around for exactly this).
A **reorder buffer** collects each transaction's changes in memory (spilling large ones to disk, or
streaming them early with `streaming = on`) and passes them to the output plugin only when the
transaction commits; aborted transactions are discarded.

For an UPDATE or DELETE, the WAL contains the replica identity columns of the old row — the primary key
by default, every column with `FULL`, or the columns of a chosen unique index — which is why that
setting determines what can be filtered and how the subscriber locates rows (by index if it has one;
otherwise by scanning, which is very slow for `FULL` without a usable index).

On the subscriber, an apply worker per subscription (plus table-sync workers for initial copies, and
parallel apply workers for large streamed transactions) applies changes as ordinary inserts, updates and
deletes — so triggers on the subscriber do not fire by default (`session_replication_role = replica`),
and constraint violations stop the worker.

## Common mistakes

- Creating a filtered or column-limited publication without checking replica identity, breaking
  writes on the publisher.
- Running DDL on the publisher first.
- Letting a stuck subscription or an unused slot retain WAL until the disk fills.
- Writing to subscriber tables that receive replicated changes.
- Forgetting sequences when using logical replication for a migration cutover.
- Using `REPLICA IDENTITY FULL` on large tables without an index the subscriber can use.

## Exercise

1. Replicate two tables from one database to another with a publication and subscription. Insert,
   update and delete on the publisher and verify each change on the subscriber.
2. Add a row filter on a column that is not in the primary key, observe the publisher-side error, and
   fix it with an appropriate replica identity.
3. Make the subscription fail by inserting a conflicting row on the subscriber; find the failing LSN in
   the log and resolve it once by fixing data and once with `ALTER SUBSCRIPTION ... SKIP`.
4. Build a minimal CDC consumer: create a logical slot with `test_decoding` (or `wal2json` if you install
   it), then write a script that polls `pg_logical_slot_get_changes` every second and prints changes for
   one table.
