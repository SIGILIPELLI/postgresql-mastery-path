---
description: "Recover to the second before a DROP TABLE: WAL archiving, verified base backups, recovery targets, timelines and a real backup policy."
---

# 05 · Point-in-Time Recovery

`pg_dump` (Level 1 · 09) gives you the database as it was when the dump ran. If the dump ran at
02:00 and someone drops the `orders` table at 15:42, a dump restore loses thirteen hours of orders.
**Point-in-time recovery (PITR)** restores to *any moment* — for example 15:41:59 — by combining a
physical **base backup** with the continuous stream of **archived WAL** since. It is the backup method
behind every serious PostgreSQL deployment and every managed service's "restore to a point in time"
button.

This lesson performs a real recovery from an accidental `DROP TABLE` on PostgreSQL 18.6.

## The two ingredients

1. **A base backup** — a physical copy of the data directory, taken while the server runs
   (`pg_basebackup`, or a tool built on the same protocol).
2. **Every WAL segment since that backup**, copied somewhere safe as each segment completes —
   **WAL archiving**.

Recovery restores the base backup, then replays archived WAL forward until the target you choose.

## Step 1 — turn on WAL archiving

```sql
ALTER SYSTEM SET archive_mode = 'on';                                         -- needs a restart
ALTER SYSTEM SET archive_command = 'test ! -f /backups/wal/%f && cp %p /backups/wal/%f';
ALTER SYSTEM SET archive_timeout = '60s';
```

```bash
pg_ctl -D $PGDATA restart -m fast
```

- `%p` is the path of the completed segment, `%f` its file name. The command must return 0 **only** if
  the file is safely stored; PostgreSQL keeps the segment in `pg_wal` and retries until it does.
- `test ! -f ... &&` refuses to overwrite an existing archive file — overwriting archived WAL is a way to
  destroy your backups silently.
- `archive_timeout` forces a segment switch at least every 60 s on a quiet server, bounding how much
  recent history could be lost if the server disappeared. (Each forced switch archives a full 16 MB
  file, so do not set it to a few seconds.)
- `cp` to a local directory is fine for a lab and wrong for production: the archive must be on other
  hardware. Real setups use tools such as pgBackRest, Barman or WAL-G, which add compression,
  parallelism, retention, encryption and object-storage support (not run in this course). PostgreSQL 15+
  also supports `archive_library` modules instead of a shell command.

Check it is working:

```sql
SELECT pg_switch_wal();
SELECT archived_count, last_archived_wal, failed_count FROM pg_stat_archiver;
```

```text
 archived_count |    last_archived_wal     | failed_count
----------------+--------------------------+--------------
              1 | 000000010000000200000054 |            0
```

`failed_count` rising means archiving is broken and WAL is piling up in `pg_wal` — alert on it, and on
the age of `last_archived_time`.

## Step 2 — take a base backup

```sql
CREATE TABLE pitr_orders (id int PRIMARY KEY, item text, created_at timestamptz DEFAULT clock_timestamp());
INSERT INTO pitr_orders (id, item) VALUES (1, 'before backup');
```

```bash
pg_basebackup -h localhost -p 54329 -U postgres -D base_backup -X stream -c fast
pg_verifybackup base_backup
```

```text
real 5.7 s
backup successfully verified
```

`-c fast` requests an immediate checkpoint instead of waiting for the next scheduled one. The backup
contains a `backup_label` (where recovery must start) and a `backup_manifest` listing every file with
a checksum, which `pg_verifybackup` checks — verify every backup you take.

PostgreSQL 17 added **incremental** base backups (`pg_basebackup --incremental=<previous manifest>`,
with `summarize_wal = on`), combined later with `pg_combinebackup`; they reduce the cost of frequent
backups of large, slowly changing databases. Not exercised here.

## Step 3 — the accident

```sql
INSERT INTO pitr_orders (id, item) VALUES (2, 'after backup, before accident'), (3, 'also before accident');
-- safe point: 2026-10-11 12:55:01.889737+05:30
DROP TABLE pitr_orders;
-- accident at: 2026-10-11 12:55:03.90574+05:30
CREATE TABLE after_accident (note text);
INSERT INTO after_accident VALUES ('written after the drop');
```

Rows 2 and 3 exist only in WAL — they were written after the base backup. A `pg_dump` from before the
accident would not have them either, unless it ran in the last few seconds.

In a real incident you rarely know the exact time. Find it from the application logs, from the server
log if `log_statement = 'ddl'` was on (a good idea), or by searching WAL with `pg_waldump` for the
transaction that dropped the table.

## Step 4 — recover to the safe point

Never recover over the damaged server; restore to a new location and copy the data back, or switch to
it once you have verified it. Here: a copy of the base backup started on port 54331.

```bash
cp -R base_backup restored
cat >> restored/postgresql.auto.conf <<'EOF'
port = 54331
archive_mode = 'off'
restore_command = 'cp /backups/wal/%f %p'
recovery_target_time = '2026-10-11 12:55:01.889737+05:30'
recovery_target_action = 'promote'
EOF
touch restored/recovery.signal
pg_ctl -D restored -l restored.log start
```

- `recovery.signal` tells the server to perform targeted archive recovery rather than a normal start
  (a standby uses `standby.signal` instead).
- `restore_command` is the inverse of `archive_command`.
- `archive_mode = 'off'` stops the restored server archiving into the same archive as the original — two
  servers writing to one archive is a classic way to corrupt it.
- `recovery_target_action = 'promote'` opens the server for writes on reaching the target; `pause`
  (the default) lets you inspect it read-only first and continue with `pg_wal_replay_resume()`, which is
  safer when you are unsure of the time.

The log tells the story:

```text
LOG:  restored log file "000000010000000200000056" from archive
LOG:  starting point-in-time recovery to 2026-10-11 12:55:01.889737+05:30
LOG:  restored log file "000000010000000200000057" from archive
LOG:  database system is ready to accept read-only connections
LOG:  recovery stopping before commit of transaction 467037, time 2026-10-11 12:55:03.900753+05:30
LOG:  redo done at 2/5700ADF0 system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.02 s
LOG:  selected new timeline ID: 2
LOG:  archive recovery complete
LOG:  database system is ready to accept connections
```

Recovery replayed WAL from the archive and stopped **before the commit of transaction 467037** — the
`DROP TABLE` — because that commit's timestamp is after the target. The restored server:

```text
$ psql -p 54331 -d lab -c "SELECT id, item FROM pitr_orders ORDER BY id"
 id |             item
----+-------------------------------
  1 | before backup
  2 | after backup, before accident
  3 | also before accident

$ psql -p 54331 -d lab -c "SELECT to_regclass('after_accident') AS after_accident_exists"
 after_accident_exists
-----------------------

```

All three rows are back, and the table created after the accident does not exist — the database is
exactly as it was at the safe point. Now you decide how to merge: copy `pitr_orders` back to production
(with `pg_dump -t` from the restored server, minding lesson Level 1 · 09's caveats), and reconcile
anything written to the table's replacement after the accident.

Other target types: `recovery_target_xid`, `recovery_target_lsn`, `recovery_target_name` (a named point
created beforehand with `pg_create_restore_point('before-migration-42')` — make one before risky
migrations), and `recovery_target = 'immediate'` (stop as soon as the backup is consistent).

## Timelines

Like promotion in lesson 3, finishing recovery started **timeline 2**. Its history file records where it
branched from timeline 1. If you later need to recover again, timelines let PostgreSQL follow the right
branch through the archive — the original server's timeline 1 continued after 12:55:03 with
`after_accident`, while the restored server's timeline 2 branched at 12:55:01. That is also why the
restored server must not archive into the same place unsupervised: its history and the original's now
diverge.

## A backup policy that actually works

- **Base backups** on a schedule (daily or weekly, depending on size and how long you can spend
  replaying WAL), plus **continuous WAL archiving** to separate storage, ideally in another region.
- **Retention** defined in terms of recovery points: keep every WAL file needed back to the oldest base
  backup you retain.
- **Verification**: `pg_verifybackup`, archiver monitoring, and — the only real proof — **regular test
  restores** to a point in time, timed, with a sanity query. The restore time is your real recovery time
  objective; measure it.
- **Logical dumps as well**, for portability and single-object restores.
- Document the procedure. At 3 a.m. during an incident is not the time to work out
  `recovery_target_action`.

## How It Actually Works

`pg_basebackup` asks the server (over the replication protocol) to start a backup: the server forces a
checkpoint and records its redo LSN as the backup's start in `backup_label`. It then streams every file
of the data directory. Files change while they are being copied, so the copy is internally inconsistent
— and that is fine, because recovery will replay all WAL from the start LSN, rewriting every block
changed during the copy (full-page images in WAL make torn copies harmless). The backup is consistent
from the moment replay passes the backup's end LSN ("consistent recovery state reached").

During targeted recovery, the startup process fetches each segment with `restore_command`, replays
records, and before applying each commit or abort record, compares its timestamp (or XID, LSN, name)
with the target. With the default `recovery_target_inclusive = on`, a transaction committing exactly at
the target is included; the first one past it is not. Then the server writes an end-of-recovery record,
picks the next timeline ID, writes the `.history` file, and opens for business.

## Common mistakes

- An `archive_command` that returns success without making sure the file is stored, or that overwrites
  existing files.
- WAL archive on the same disk or server as the database.
- Never testing a restore; discovering a gap in the archive during an incident.
- Recovering on top of the damaged cluster instead of into a new directory.
- Letting the restored server archive into the original's archive.
- Not knowing the incident time and promoting before checking (`recovery_target_action = 'pause'`).

## Exercise

1. Enable archiving on your lab server, take a base backup, then create, change and drop data over a few
   minutes, noting the times. Restore three times to three different targets and confirm each state.
2. Create a named restore point before a "risky migration", perform the migration, and recover to the
   restore point by name.
3. Restore with `recovery_target_action = 'pause'`, inspect the data read-only, then use
   `pg_wal_replay_resume()` to continue to a later target.
4. Delete one archived WAL file from the middle of your archive (on a copy!) and attempt recovery past it.
   Record exactly what the log says — this is what a broken archive looks like.
