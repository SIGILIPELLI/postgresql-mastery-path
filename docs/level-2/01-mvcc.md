---
description: "Watch MVCC happen: xmin, xmax and ctid on real row versions, pageinspect output, snapshots, and why an UPDATE is an insert plus a delete."
---

# 01 · MVCC: How Rows Become Visible

In Level 1 · 07, two sessions looked at the same row and saw different values. That was not a
trick of caching — both versions of the row physically existed on disk at the same time.
PostgreSQL's concurrency model, **multi-version concurrency control (MVCC)**, keeps old row versions
around so that readers never block writers and writers never block readers. Every behaviour in the
rest of Level 2 — VACUUM, bloat, HOT updates, index-only scans, transaction-ID wraparound — follows
from this one design decision.

This lesson shows MVCC directly, using hidden system columns and the `pageinspect` extension to
read raw table pages. Outputs come from PostgreSQL 18.6. Your transaction IDs will be different
numbers; the relationships between them will be the same.

## Hidden columns: xmin, xmax, ctid

Every table has system columns that `SELECT *` does not show:

- **`xmin`** — the ID of the transaction that created this row version.
- **`xmax`** — the ID of the transaction that deleted or replaced it (0 if none, or if that
  transaction was rolled back… mostly; see below).
- **`ctid`** — the physical location of this version: `(page, item)`.

```sql
CREATE EXTENSION pageinspect;            -- superuser only; never needed in application code
CREATE TABLE mv (id int PRIMARY KEY, val text);
INSERT INTO mv VALUES (1, 'apple'), (2, 'banana');
SELECT ctid, xmin, xmax, * FROM mv;
```

```text
 ctid  |  xmin  | xmax | id |  val
-------+--------+------+----+--------
 (0,1) | 101331 |    0 |  1 | apple
 (0,2) | 101331 |    0 |  2 | banana
```

Both rows were created by transaction 101331 and sit in slots 1 and 2 of page 0.

## An UPDATE writes a new row

```sql
UPDATE mv SET val = 'apricot' WHERE id = 1;
SELECT ctid, xmin, xmax, * FROM mv;
```

```text
 ctid  |  xmin  | xmax | id |   val
-------+--------+------+----+---------
 (0,2) | 101331 |    0 |  2 | banana
 (0,3) | 101333 |    0 |  1 | apricot
```

Row 1 has **moved** to slot 3 and has a new `xmin`. Nothing was changed in place. Look at the raw
page to see what happened to the old version:

```sql
SELECT lp, t_xmin, t_xmax, t_ctid, t_data
FROM heap_page_items(get_raw_page('mv', 0));
```

```text
 lp | t_xmin | t_xmax | t_ctid |           t_data
----+--------+--------+--------+----------------------------
  1 | 101331 | 101333 | (0,3)  | \x010000000d6170706c65
  2 | 101331 |      0 | (0,2)  | \x020000000f62616e616e61
  3 | 101333 |      0 | (0,3)  | \x010000001161707269636f74
```

Slot 1 still holds `apple` (`6170706c65` is "apple" in hex). Its `t_xmax` is 101333 — "deleted by
the updating transaction" — and its `t_ctid` points forward to `(0,3)`, the new version. An UPDATE
in PostgreSQL is an insert of a new version plus marking the old one as expired. A DELETE is just
the marking.

This is the root of several facts you will meet repeatedly:

- Updating one column of a wide row writes a **complete new copy** of the row.
- Old versions take up space until something removes them. That something is **VACUUM** (next
  lesson).
- Indexes point at row versions by `ctid`. A new version at a new location may need new index
  entries — unless the update qualifies as **HOT** (lesson 3).
- `ctid` is not a stable row identifier. Never store it.

## A rolled-back delete

```sql
BEGIN;
DELETE FROM mv WHERE id = 2;
SELECT pg_current_xact_id();      -- 101334
ROLLBACK;
SELECT ctid, xmin, xmax, * FROM mv;
```

```text
 ctid  |  xmin  |  xmax  | id |   val
-------+--------+--------+----+---------
 (0,2) | 101331 | 101334 |  2 | banana
 (0,3) | 101333 |      0 |  1 | apricot
```

The banana row is visible — the delete was rolled back — yet its `xmax` still says 101334. A
rollback in PostgreSQL does not go back and undo anything; it just records "transaction 101334
aborted" in the commit log (`pg_xact`):

```sql
SELECT pg_xact_status(xmax::text::xid8) FROM mv WHERE id = 2;
```

```text
 pg_xact_status
----------------
 aborted
```

Readers who see `xmax = 101334` check that status and conclude the deletion never happened. This
is why `ROLLBACK` is instant no matter how much work the transaction did, and why the inverse is
also true: a rolled-back bulk `INSERT` of ten million rows leaves ten million dead rows behind for
VACUUM to clean up.

## Snapshots: deciding what you can see

Two sessions. A updates a row and has not committed; B reads it:

```text
A's xid: 101336
A sees: [('(0,4)', '101336', '0', 'blueberry')]
B's snapshot: 101336:101336:
B sees: [('(0,2)', '101331', '101336', 'banana')]
after A commits, B sees: [('(0,4)', '101336', '0', 'blueberry')]
```

Both versions exist. A sees its own new version (0,4). B sees the old version (0,2) with
`xmax = 101336`, because transaction 101336 is not visible in B's snapshot.

A snapshot, printed by `pg_current_snapshot()`, has the form `xmin:xmax:xip_list`:

- **xmin** — every transaction below this had finished when the snapshot was taken.
- **xmax** — one past the newest *completed* transaction; every ID at or above it is treated as
  still running (or not started yet).
- **xip_list** — IDs between the two that were still in progress.

Here `101336:101336:` says "everything below 101336 is finished; 101336 and later are invisible".
A row version is visible to a snapshot when its `xmin` committed and is visible to the snapshot,
and its `xmax` is either empty, aborted, or not visible to the snapshot.

## Commit status hint bits

Checking `pg_xact` for every row on every read would be slow, so the first reader to discover a
transaction's outcome sets **hint bits** in the tuple header. In the earlier `pageinspect` output,
the `t_infomask` for slot 1 had the "xmin committed" bit set (I selected it as
`xmin_committed_hint = t`), while the brand-new rows did not yet. Two side effects matter in
practice:

- The **first `SELECT` after a large load can write to disk**, because it sets hint bits on every
  page and dirties them.
- Hint-bit changes are not normally WAL-logged, but with data checksums enabled (the default for
  new clusters since PostgreSQL 18) setting them can generate extra WAL in the form of full-page
  images.

## Transaction IDs are 32 bits

`pg_current_xact_id()` returns a 64-bit value (`xid8`) that includes an epoch, but the `xmin` and
`xmax` stored in each tuple are **32-bit**. About 4 billion transactions wrap around, and comparisons
use modulo arithmetic: from any transaction's point of view, roughly 2 billion IDs are in the past
and 2 billion in the future. A row that is old enough would suddenly look like it was created "in
the future" and vanish. VACUUM prevents this by **freezing** old rows — marking them as visible to
everyone regardless of `xmin` — and the next lesson shows how to monitor it. It is the reason
VACUUM is not optional.

Read-only transactions do not consume an ID at all; one is assigned only on the first write. Plain
`SELECT`-only traffic cannot cause wraparound.

## How It Actually Works

A heap page is 8 kB: a 24-byte page header, an array of 4-byte **line pointers** growing from the
front, tuples growing from the back, and free space in between. A `ctid` of `(0,3)` means page 0,
line pointer 3. Indexes store `ctid`s, and the line-pointer indirection lets PostgreSQL move a tuple
within a page (to defragment it) without touching indexes.

Each tuple starts with a 23-byte header: `t_xmin`, `t_xmax`, `t_cid` (command ID, so a statement
does not see rows inserted by itself later in the same transaction), `t_ctid` (pointer to the newer
version, or to itself if latest), `t_infomask` and `t_infomask2` (flag bits: has nulls, hint bits,
locked-only xmax, HOT-updated, …) and the null bitmap. Then the column data.

When the executor reads a tuple, the visibility function compares its `xmin`/`xmax` with the current
snapshot, consults hint bits, falls back to `pg_xact` (the commit log: 2 bits of status per
transaction ID) and to the shared list of running transactions. The cost of MVCC is paid in space
and in this per-tuple check, rather than in locks.

## Common mistakes

- Believing `UPDATE` changes a row in place, and being surprised that a table doubles in size after
  `UPDATE big_table SET flag = true`.
- Using `ctid` as a key in application code. It changes on every update and after `VACUUM FULL`.
- Rolling back huge transactions and expecting the space back immediately.
- Holding a transaction open for hours: its snapshot keeps every row version it might need,
  preventing cleanup across the whole database (next lesson).
- Assuming `SELECT` never writes. Hint bits and pruning can make reads dirty pages.

## Exercise

1. Create a table with one row. Update it five times, then use `heap_page_items` to list every
   version and follow the `t_ctid` chain from the oldest to the newest.
2. In two `psql` sessions, open a Repeatable Read transaction in B, then update and commit in A.
   Show that B still reads the old version and that both versions are on the page.
3. Insert 100,000 rows in a transaction and roll it back. Check the table size with
   `pg_relation_size`. Where did the space go, and what will reclaim it?
4. Use `pg_current_snapshot()` while another session holds an open write transaction, and identify
   that transaction in the snapshot.
