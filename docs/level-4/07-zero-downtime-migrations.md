---
description: "Schema changes without outages: which ALTERs rewrite or block, measured lock impact, lock_timeout, NOT VALID and expand/contract."
---

# 07 · Zero-Downtime Schema Migrations

Most schema changes are instant. A few rewrite the whole table while holding an `ACCESS EXCLUSIVE`
lock, blocking every read and write for the duration — on a large table, a self-inflicted outage.
And even an instant change can cause one if it waits in the lock queue behind a long transaction
(Level 2 · 08). The skill this lesson teaches is knowing which is which, and the patterns for doing the
dangerous ones safely.

All measurements are on the 1-million-row, 85 MB `orders` table from Level 2 (with nine indexes), on
PostgreSQL 18.6 on a laptop. A real table may be a thousand times larger, and every "seconds" below
becomes "hours".

## Measuring what a migration blocks

To see the impact rather than guess it, a probe thread ran a single-row `UPDATE` every 50 ms while each
DDL statement executed, and recorded the slowest probe:

```text
ALTER TABLE orders ALTER COLUMN customer_id TYPE int           took  6.48 s; worst concurrent single-row UPDATE  6263.1 ms
ALTER TABLE orders ADD CONSTRAINT total_nonneg CHECK (total >= took  0.01 s; worst concurrent single-row UPDATE     2.5 ms
ALTER TABLE orders VALIDATE CONSTRAINT total_nonneg            took  0.12 s; worst concurrent single-row UPDATE    10.7 ms
CREATE INDEX orders_channel_idx ON orders (channel)            took  0.18 s; worst concurrent single-row UPDATE   155.0 ms
DROP INDEX orders_channel_idx                                  took  0.01 s; worst concurrent single-row UPDATE     2.7 ms
CREATE INDEX CONCURRENTLY orders_channel_idx ON orders (channe took  0.22 s; worst concurrent single-row UPDATE     1.6 ms
```

- The column type change blocked a one-row update for **6.3 seconds** — the whole rewrite.
- Plain `CREATE INDEX` blocked writes for its whole build (its SHARE lock allows reads but not writes).
- `CREATE INDEX CONCURRENTLY` and `VALIDATE CONSTRAINT` let writes continue.

The probe idea is worth stealing: run your migration against a production-sized copy while a script
measures what it blocks.

## Instant, rewrite, or scan?

```text
ALTER TABLE orders ADD COLUMN channel text NOT NULL DEFAULT 'web';             Time: 3.937 ms
ALTER TABLE orders ADD COLUMN import_batch uuid DEFAULT gen_random_uuid();     Time: 6345.205 ms
```

Since PostgreSQL 11, adding a column with a **constant** default is instant: the default is stored in
the catalog and returned for old rows that lack the column. A **volatile** default (`gen_random_uuid()`,
`clock_timestamp()`) must be computed per row, so the table is rewritten — the file node changed
(`29581 → 31319`), proving a new copy was written. If you need per-row values, add the column with no
default (or a constant) and backfill in batches.

| Operation | Effect |
|-----------|--------|
| `ADD COLUMN` (nullable, or constant default) | instant |
| `ADD COLUMN ... DEFAULT volatile()` | rewrite |
| `DROP COLUMN` | instant (space reclaimed later) |
| `ALTER COLUMN TYPE` (most changes) | rewrite + rebuild indexes on it |
| `ALTER COLUMN TYPE varchar(n)` → `text`, or larger `varchar(n)` | instant (binary compatible) |
| `SET NOT NULL` | full scan under ACCESS EXCLUSIVE — unless a validated `CHECK (col IS NOT NULL)` exists |
| `ADD CONSTRAINT ... CHECK / FOREIGN KEY` | full scan under lock — use `NOT VALID` + `VALIDATE` |
| `ADD PRIMARY KEY / UNIQUE` | index build under lock — build the index `CONCURRENTLY` first, then `ADD CONSTRAINT ... USING INDEX` |
| `CREATE INDEX` | blocks writes — use `CONCURRENTLY` |
| `RENAME` table/column | instant, but breaks code that uses the old name |

Measured examples from the table:

```text
ALTER TABLE orders ALTER COLUMN customer_id TYPE bigint;       Time: 6080.988 ms  (rewrite)
ALTER TABLE orders ALTER COLUMN status TYPE varchar(20);       Time: 8057.236 ms  (scan + index rebuilds)
ALTER TABLE orders ALTER COLUMN status TYPE text;              Time: 54.331 ms    (binary compatible)
```

## Pattern 1: always set lock_timeout

Every migration statement should give up quickly rather than queue behind a long transaction and block
everyone behind it:

```sql
SET lock_timeout = '2s';
SET statement_timeout = '30s';   -- for the instant steps; raise it for validations and backfills
ALTER TABLE orders ADD COLUMN channel text;
```

If it fails with `55P03 canceling statement due to lock timeout`, retry after a short pause. Several
schema-migration tools can do this for you; your own scripts should too (exercise 3).

## Pattern 2: NOT NULL without a long lock

```text
ALTER TABLE orders ADD CONSTRAINT region_not_null CHECK (region IS NOT NULL) NOT VALID;   Time: 23.175 ms
ALTER TABLE orders VALIDATE CONSTRAINT region_not_null;                                   Time: 2514.749 ms
ALTER TABLE orders ALTER COLUMN region SET NOT NULL;                                      Time: 3.526 ms
ALTER TABLE orders DROP CONSTRAINT region_not_null;                                       Time: 5.561 ms
```

The validation scan takes the time but only a `SHARE UPDATE EXCLUSIVE` lock (reads and writes
continue). `SET NOT NULL` then sees the validated CHECK constraint proves no NULLs exist and skips its
own scan, so its exclusive lock lasts milliseconds. PostgreSQL 18 shortens this further with
`ALTER TABLE ... ADD CONSTRAINT name NOT NULL col NOT VALID` followed by `VALIDATE CONSTRAINT`
(Level 1 · 06).

## Pattern 3: expand, backfill, contract

For changes that would rewrite the table — changing a column's type, splitting a column, renaming
something the application uses — do it in phases, each of which is safe on its own. Changing
`customer_id` from `int` to `bigint`:

**Expand** — a new column, kept in sync for new writes:

```sql
ALTER TABLE orders ADD COLUMN customer_id_big bigint;

CREATE FUNCTION orders_sync_customer_id() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN NEW.customer_id_big := NEW.customer_id; RETURN NEW; END $$;
CREATE TRIGGER orders_sync_customer_id BEFORE INSERT OR UPDATE OF customer_id ON orders
  FOR EACH ROW EXECUTE FUNCTION orders_sync_customer_id();
```

**Backfill** existing rows in batches, committing between them (Level 3 · 03):

```sql
CREATE PROCEDURE backfill_customer_id_big(batch int DEFAULT 100000) LANGUAGE plpgsql AS $$
DECLARE last_id bigint := 0; max_id bigint;
BEGIN
  SELECT max(id) INTO max_id FROM orders;
  WHILE last_id < max_id LOOP
    UPDATE orders SET customer_id_big = customer_id
    WHERE id > last_id AND id <= last_id + batch AND customer_id_big IS NULL;
    last_id := last_id + batch;
    COMMIT;
  END LOOP;
END $$;
CALL backfill_customer_id_big();
```

```text
Time: 28325.471 ms (00:28.325)
 missing
---------
       0
```

The backfill took **28 seconds — over four times longer than the 6-second rewrite** it replaces. That is
the trade: much more total work (one million updates, each writing new versions to a table with nine
indexes), but never more than one 100,000-row batch of row locks held at a time and no table-wide lock.
In production, add a pause between batches and watch replication lag (lesson 3).

**Contract** — make the new column authoritative in short transactions:

```sql
ALTER TABLE orders ADD CONSTRAINT cid_big_nn CHECK (customer_id_big IS NOT NULL) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT cid_big_nn;
BEGIN;
SET LOCAL lock_timeout = '2s';
ALTER TABLE orders ALTER COLUMN customer_id_big SET NOT NULL;
ALTER TABLE orders DROP CONSTRAINT cid_big_nn;
DROP TRIGGER orders_sync_customer_id ON orders;
ALTER TABLE orders RENAME COLUMN customer_id TO customer_id_old;
ALTER TABLE orders RENAME COLUMN customer_id_big TO customer_id;
COMMIT;
```

Each statement in the final transaction took under a millisecond. But `\d orders` afterwards revealed a
mistake in my first run of this procedure:

```text
 customer_id_old | integer                  |           | not null |
 customer_id     | bigint                   |           | not null |
    "orders_customer_created_idx" btree (customer_id_old, created_at DESC)
    "orders_customer_idx" btree (customer_id_old)
    "orders_customer_incl_idx" btree (customer_id_old) INCLUDE (total)
```

**The indexes stayed on the old column.** Indexes (and foreign keys, and views, and statistics) belong to
the column they were created on, and they follow it through a rename. Every query on `customer_id` would
now be a sequential scan. The expand phase must also `CREATE INDEX CONCURRENTLY` matching indexes on the
new column, and foreign keys must be re-added with `NOT VALID` + `VALIDATE` — all before the swap. Then
drop `customer_id_old` and its indexes in a later release, once nothing reads it.

### Coordinating with application deploys

Renames and type changes also need the application to cooperate. The general sequence:

1. Migration: expand (new column/table, sync trigger, backfill, indexes).
2. Deploy: application reads the new column (or both), writes both.
3. Migration: contract the database side.
4. Deploy: application stops referring to the old column.
5. Migration, a release later: drop the old column.

Never ship a migration that breaks the *currently running* application version: during a rolling deploy
both old and new versions run against the same schema.

## Long-running migrations and replicas

Large backfills and index builds generate WAL that standbys must replay (watch lag), hold back VACUUM if
run in one long transaction (batch them), and `CREATE INDEX CONCURRENTLY` waits for older transactions
(lesson 6 showed it stuck behind an idle-in-transaction session). If a concurrent build fails, it leaves
an `INVALID` index:

```sql
SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;   -- drop and retry these
```

## How It Actually Works

`ALTER TABLE` works out, for the combination of subcommands given, the strongest lock it needs and
whether it must rewrite (copy every row into a new relfilenode, as the changing file node showed), scan
(verify rows without rewriting), or only update the catalog. Rewrites rebuild every index on the table
too. Some type changes are flagged *binary coercible* in `pg_cast` (`varchar` → `text`), so no rewrite is
needed; constraints are checked by scanning when they cannot be proven from existing ones.

A "fast default" for a new column is stored in `pg_attribute.attmissingval`; when a tuple physically has
fewer attributes than the table definition, the executor fills the missing ones from it, which is why
adding the column is instant and why the next rewrite of the table materialises the value.

## Common mistakes

- DDL without `lock_timeout` on busy tables.
- Volatile defaults on `ADD COLUMN` for large tables.
- Changing column types in place on large tables.
- `CREATE INDEX` instead of `CREATE INDEX CONCURRENTLY`; not checking for invalid indexes afterwards.
- Forgetting indexes, foreign keys and dependent views when swapping columns.
- Migrations that are incompatible with the application version still running.

## Exercise

1. On a copy of a large table, run each operation from the table above with the probe script running and
   record the worst blocked write. Which surprised you?
2. Complete the expand/contract `int` → `bigint` migration properly: concurrent indexes on the new column,
   foreign keys re-added safely, the old column dropped. Verify every query plan that used the old
   indexes.
3. Write a migration runner that wraps each statement with `lock_timeout = '2s'` and retries up to five
   times with back-off on SQLSTATE 55P03.
4. Use `pg_stat_progress_create_index` to watch a concurrent index build on a 10-million-row table while a
   long transaction is open in another session. Which phase does it wait in?
