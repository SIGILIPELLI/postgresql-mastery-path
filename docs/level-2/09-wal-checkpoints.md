---
description: "Write-ahead logging in PostgreSQL: LSNs, WAL records, full-page images after checkpoints, checkpoint tuning, synchronous_commit and fsync."
---

# 09 · WAL, Checkpoints & Durability

When `COMMIT` returns, PostgreSQL promises your data survives a crash — power cut, kernel panic,
`kill -9`. Yet the changed table pages are still sitting in memory and will be written to their
files minutes later. The mechanism that squares that circle is the **write-ahead log (WAL)**, and it
is also the foundation of replication (Level 4 · 03/04) and point-in-time recovery (Level 4 · 05).
This lesson looks at real WAL records and the settings that trade durability against speed.

Output is from PostgreSQL 18.6 on macOS.

## The rule: log first, data later

Before a changed data page may be written to disk, the WAL record describing the change must be on
disk. And at `COMMIT`, the WAL up to and including the commit record is flushed (`fsync`ed). That is
all durability requires: after a crash, the server replays WAL from the last checkpoint and re-applies
every change whose data page did not make it to disk.

Writing WAL is cheap compared with writing data pages: it is sequential, it contains only the
change, and many transactions' records are flushed together.

## Positions in the log: LSNs

Every byte of WAL has a **log sequence number** (LSN), a 64-bit position shown as two hex halves:

```sql
CHECKPOINT;
SELECT pg_current_wal_lsn() AS before \gset
INSERT INTO walt SELECT g, 'row ' || g FROM generate_series(1, 1000) g;
SELECT pg_current_wal_lsn() AS after \gset
SELECT :'before' AS before_lsn, :'after' AS after_lsn,
       pg_size_pretty(pg_wal_lsn_diff(:'after', :'before')) AS wal_for_first_insert;
```

```text
 before_lsn | after_lsn  | wal_for_first_insert
------------+------------+----------------------
 0/71B4AEE0 | 0/71B64798 | 102 kB
```

`pg_wal_lsn_diff` is how you measure WAL volume, replication lag in bytes, and how far a backup
has progressed. WAL is stored in 16 MB segment files in `pg_wal/`, named after the position:

```text
SELECT pg_walfile_name(pg_current_wal_lsn());
 000000010000000000000071
```

(timeline 1, segment 0x71).

## Full-page images: why the first change after a checkpoint is expensive

```sql
CHECKPOINT;
UPDATE walt SET v = 'after checkpoint' WHERE id = 500;   -- measured: 12 kB of WAL
UPDATE walt SET v = 'same page again' WHERE id = 501;    -- measured: 192 bytes of WAL
```

```text
 first_change_after_checkpoint | second_change_same_page
-------------------------------+-------------------------
 12 kB                         | 192 bytes
```

Same kind of change, 60× more WAL. `pg_waldump` shows why:

```text
$ pg_waldump -p $PGDATA/pg_wal -s 0/71B649E8 -e 0/71B67908
rmgr: Heap   len (rec/tot): 59/8223, tx: 101442, desc: LOCK xmax: 101442, off: 130, ... blkref #0: rel 1663/24681/29772 blk 2 FPW
rmgr: Heap   len (rec/tot): 100/3572, tx: 101442, desc: UPDATE old_xmax: 101442, old_off: 130, ... new_off: 78, blkref #0: ...
rmgr: Transaction len (rec/tot): 46/46, tx: 101442, desc: COMMIT 2026-10-11 12:20:59.111040 IST
rmgr: Heap2  len (rec/tot): 56/56, tx: 0, desc: PRUNE_ON_ACCESS snapshotConflictHorizon: 101442, ... dead: [130]
rmgr: Heap   len (rec/tot): 86/86, tx: 101443, desc: HOT_UPDATE old_xmax: 101443, old_off: 131, ... new_off: 186
rmgr: Transaction len (rec/tot): 46/46, tx: 101443, desc: COMMIT 2026-10-11 12:20:59.111390 IST
```

(Lines shortened.) The first record is 59 bytes of change plus a whole **8 kB page image** (`FPW`,
total 8,223 bytes). Because the full page did not fit, the update also moved the row to another page,
and that page's image came along too (3,572 bytes — the free "hole" in a page is not logged). The
second update is an 86-byte `HOT_UPDATE` (lesson 3) — and you can even see the opportunistic
pruning record from lesson 2 (`PRUNE_ON_ACCESS`).

Why page images? The OS writes an 8 kB page as several smaller disk blocks. If power fails mid-write,
the page on disk is **torn**: half old, half new. A WAL record saying "change bytes 130–160" cannot be
applied to a torn page. So after each checkpoint, the first modification of every page logs the
whole page; recovery restores the image and replays later changes on top. This is
`full_page_writes = on` — never turn it off unless your storage guarantees atomic 8 kB writes.

The consequence: **more frequent checkpoints mean more WAL**, because more "first changes after a
checkpoint" happen. `wal_compression` (`lz4` or `zstd` where supported) compresses those images and
is usually a cheap win on write-heavy systems.

## Checkpoints

A checkpoint writes every dirty buffer to its data file, fsyncs the files, and records a **redo
point**: recovery only needs WAL from there. Settings:

```text
             name             | setting | unit
------------------------------+---------+------
 checkpoint_completion_target | 0.9     |
 checkpoint_timeout           | 300     | s
 max_wal_size                 | 1024    | MB
 min_wal_size                 | 80      | MB
```

A checkpoint starts when `checkpoint_timeout` elapses **or** WAL since the last one approaches
`max_wal_size`, whichever comes first. `checkpoint_completion_target = 0.9` spreads the writes over
90% of the interval instead of flooding the disk.

Checkpoints triggered by WAL volume ("requested") rather than time are a sign `max_wal_size` is too
small for the write load:

```sql
SELECT num_timed, num_requested, write_time, sync_time, buffers_written FROM pg_stat_checkpointer;
-- this lab: (2, 10, 431167.0, 191.0, 28041) — mostly requested, from the manual CHECKPOINTs and bulk loads in these lessons
```

On a production server you want nearly all checkpoints timed. Typical tuning: `checkpoint_timeout`
of 15 minutes and `max_wal_size` large enough (often several GB to tens of GB) that timed checkpoints
dominate. The trade-off is crash-recovery time: after a crash, up to one checkpoint interval of WAL
must be replayed. (`pg_stat_checkpointer` is PostgreSQL 17+; older versions have these columns in
`pg_stat_bgwriter`.)

## synchronous_commit: trading durability for latency

```text
synchronous_commit=on   3000 commits in 0.21 s  (70 µs per commit)
synchronous_commit=off  3000 commits in 0.12 s  (41 µs per commit)
synchronous_commit=on   3000 commits in 0.22 s  (73 µs per commit)
synchronous_commit=off  3000 commits in 0.12 s  (40 µs per commit)
```

With `synchronous_commit = off`, `COMMIT` returns before its WAL is flushed; the WAL writer flushes
it within about three times `wal_writer_delay` (200 ms). A crash can lose the last fraction of a
second of **acknowledged** commits — but never corrupts the database or leaves a transaction
half-applied. You can set it per transaction:

```sql
BEGIN;
SET LOCAL synchronous_commit = off;
INSERT INTO page_views ...;   -- losing a few of these in a crash is acceptable
COMMIT;
```

That makes it a precise tool: keep payments fully durable and relax only for data you can afford to
lose. With replication, the same setting gains extra values (`remote_write`, `remote_apply`) that
wait for standbys (Level 4 · 03).

### A warning about these numbers

70 µs per durable commit is implausibly fast for real disk flushes, and the reason matters.
`pg_test_fsync` on the same Mac:

```text
        open_datasync                     21443.884 ops/sec      47 usecs/op
        fdatasync                         43006.325 ops/sec      23 usecs/op
        fsync                             42136.680 ops/sec      24 usecs/op
        fsync_writethrough                  251.185 ops/sec    3981 usecs/op
```

On macOS, ordinary `fsync` hands data to the drive but does not force the drive to empty its volatile
cache; only `F_FULLFSYNC` (`fsync_writethrough`) does, and it costs about 4 ms here. The server was
using `wal_sync_method = open_datasync`, so a power cut could still lose commits it had acknowledged.
That is fine for a laptop lab — and a reminder to run `pg_test_fsync` on real servers, and to trust
storage only if it honours flushes (or has power-loss protection). On Linux the default `fdatasync`
does request a cache flush from the device.

Never set `fsync = off` on a database you care about: unlike `synchronous_commit = off`, a crash
with `fsync` off can corrupt the database.

## wal_level

```text
 wal_level                    | logical
```

`minimal` logs only what crash recovery needs (and can skip WAL for some bulk operations), `replica`
(the default) adds what physical replication and archiving need, `logical` adds information for
logical decoding (Level 4 · 04). The cluster used to write this course was created with `logical` from
the start, to be ready for that lesson; the setting needs a restart to change.

## How It Actually Works

Backends do not write WAL files directly. A backend inserting a record reserves space in the shared
**WAL buffers** (`wal_buffers`), copies the record in, and continues. At commit it asks for the WAL
up to its commit record to be flushed; if another backend's flush already covered that LSN, it
returns immediately — this **group commit** effect is why many concurrent commits cost far less than
one fsync each. The WAL writer process flushes in the background for asynchronous commits and to keep
buffers from filling.

Each data page header stores the LSN of the last WAL record that modified it (`pd_lsn`, lesson 3).
Before the buffer manager writes a dirty page to its file, it ensures WAL is flushed at least to that
LSN — the "write-ahead" rule, enforced per page.

During crash recovery, the startup process reads `pg_control` to find the last checkpoint's redo
point and replays records sequentially. For each record touching a page, it reads the page and
compares the page's LSN with the record's: if the page is already newer, the change is skipped; if
the record carries a full-page image, the image simply replaces the page. That makes replay
idempotent, which is exactly what both streaming replication and point-in-time recovery reuse —
a standby is a server permanently in recovery, applying WAL as it arrives.

## Common mistakes

- `fsync = off` or `full_page_writes = off` "for speed" on real data.
- `max_wal_size` left at 1 GB on a write-heavy server, causing frequent requested checkpoints and
  extra full-page-image WAL.
- Manually deleting files in `pg_wal` to free disk.
- Using `synchronous_commit = off` globally when only some data can tolerate loss.
- Benchmarking commit latency on a laptop or on storage that ignores flushes, and assuming production
  will match.

## Exercise

1. Measure the WAL generated by inserting 100,000 rows with single-row autocommit inserts versus one
   multi-row `INSERT ... SELECT`. Explain the difference using `pg_waldump --stats`.
2. Run `CHECKPOINT`, then update one row in each of 1,000 different pages, and measure WAL. Repeat the
   same updates without a checkpoint in between. Then enable `wal_compression = lz4` (if your build
   supports it) and measure again.
3. Run `pg_test_fsync` on your own machine and decide which `wal_sync_method` is both durable and
   fastest.
4. Use `SET LOCAL synchronous_commit = off` inside a transaction and confirm with `SHOW` that it reverts
   afterwards.
