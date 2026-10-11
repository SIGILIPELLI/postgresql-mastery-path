---
description: "Build streaming replication: pg_basebackup, replication slots, monitoring lag, synchronous commit, slot WAL retention and promotion."
---

# 03 · Streaming Replication

A **standby** is a copy of the primary server that continuously replays the primary's WAL
(Level 2 · 09). It gives you a hot spare to fail over to, a place to run read-only queries, and the
basis for zero-data-loss configurations. Physical streaming replication is built in, copies the whole
cluster byte for byte, and is the foundation of almost every high-availability PostgreSQL setup.

Rather than describe it, this lesson builds one: a primary on port 54329 and a standby on 54330, both
PostgreSQL 18.6 on one laptop. Running both on one machine is only for learning — a real standby must
be on separate hardware, ideally in a separate failure zone.

## Prerequisites on the primary

Defaults since PostgreSQL 10 already allow replication: `wal_level = replica` (this lab uses
`logical`, which includes it), `max_wal_senders = 10`, and `pg_hba.conf` needs a `replication` line for
the connecting role (the lab's trust rules cover it; in production use `scram-sha-256` or certificates):

```text
host    replication     replicator      10.0.0.0/24     scram-sha-256
```

Create a dedicated role and a **replication slot**:

```sql
CREATE ROLE replicator LOGIN REPLICATION PASSWORD 'repl-dev';
SELECT pg_create_physical_replication_slot('standby1');
```

The slot makes the primary keep any WAL the standby has not yet received, so a standby that falls
behind or restarts can always catch up. That guarantee has a cost you will see shortly.

## Cloning the primary

```bash
pg_basebackup -h localhost -p 54329 -U replicator -D standby1 -R -S standby1 -X stream -P
```

```text
3365137/3365137 kB (100%), 1/1 tablespace
real 7.6 s
```

- `-D standby1` — the new data directory (3.3 GB here: every database in this course's lab cluster).
- `-X stream` — also stream the WAL generated *during* the copy, so the copy is consistent.
- `-R` — write `standby.signal` and the connection settings, so the copy starts as a standby.
- `-S standby1` — use the slot.

`-R` wrote these into `postgresql.auto.conf` (trimmed):

```text
primary_conninfo = 'user=replicator passfile=''/Users/.../.pgpass'' host=localhost port=54329 ...'
primary_slot_name = 'standby1'
```

The copy also inherited the primary's `postgresql.conf`, including its port. Before starting it on the
same machine I set `port = 54330`, a smaller `shared_buffers`, and `hot_standby_feedback = on`
(explained below).

```bash
pg_ctl -D standby1 -l standby1.log start
```

```text
LOG:  redo starts at 1/DF000028
LOG:  completed backup recovery with redo LSN 1/DF000028 and end LSN 1/DF000120
LOG:  consistent recovery state reached at 1/DF000120
LOG:  database system is ready to accept read-only connections
LOG:  started streaming WAL from primary at 1/E0000000 on timeline 1
```

## Watching it work

On the primary:

```sql
SELECT application_name, state, sync_state,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS replay_lag_bytes,
       replay_lag
FROM pg_stat_replication;
```

```text
 application_name |   state   | sync_state | replay_lag_bytes |   replay_lag
------------------+-----------+------------+------------------+-----------------
 walreceiver      | streaming | async      | 96 bytes         | 00:00:00.000167
```

On the standby, data written to the primary appears, and writes are refused:

```text
$ psql -p 54330 -d lab -c "SELECT * FROM repl_demo"
 id |        note
----+--------------------
  1 | written on primary

$ psql -p 54330 -d lab -c "INSERT INTO repl_demo VALUES (2, 'on standby')"
ERROR:  cannot execute INSERT in a read-only transaction
```

`SELECT pg_is_in_recovery()` returns true on a standby — useful in health checks and for routing.

### Lag under load

During a 10-second write-heavy `pgbench` run on the primary (about 9,900 transactions per second),
sampled halfway through:

```text
lag during load: 83 MB / 00:00:01.563115
```

The standby was 83 MB and about 1.5 seconds of replay behind. Replication is **asynchronous** by
default: a commit returns once it is durable on the primary, and the standby catches up afterwards.
A failover at that moment could lose whatever WAL had not yet *reached* the standby. Monitor lag in
both bytes (`pg_wal_lsn_diff`) and time (`replay_lag`), and alert on it.

## Synchronous replication

```sql
ALTER SYSTEM SET synchronous_standby_names = 'walreceiver';   -- the standby's application_name
SELECT pg_reload_conf();
```

```text
walreceiver|sync
```

Now each commit waits until the standby confirms it has flushed the WAL. Same benchmark:

```text
async:  latency average = 0.807 ms   tps = 9912.837342
sync:   latency average = 0.960 ms   tps = 8336.187427
```

About 16% fewer transactions per second — with the "network" being a loopback interface. Across real
networks every commit pays at least one round trip to the standby, so the cost grows with distance.
`synchronous_commit` chooses how much the commit waits for: `remote_write` (standby received it),
`on` (flushed to its disk, the default), or `remote_apply` (replayed, so a read on the standby will see
it). Use `ANY 1 (s1, s2)` / `FIRST 1 (s1, s2)` to tolerate one standby being down.

### What happens when the only sync standby is down

I stopped the standby and inserted a row:

```text
after 3 s the session is waiting on: ('IPC', 'SyncRep')
INSERT returned after 3.0 s
WARNING: canceling wait for synchronous replication due to user request | The transaction has already committed locally, but might not have been replicated to the standby.
row visible on primary: 1
```

The commit **hung** — waiting forever for a standby that was not there — until I cancelled it after 3
seconds. And the cancellation did not undo anything: the transaction had already committed on the
primary, and the row was visible. So synchronous replication with a single standby turns "standby
down" into "primary cannot commit", and a client that times out cannot conclude its transaction failed.
Production setups use at least two synchronous candidates (`ANY 1 (...)`) and failover tooling that
manages `synchronous_standby_names`.

I reset `synchronous_standby_names` before continuing.

## The replication slot trap

With the standby still stopped, another 10-second `pgbench` run on the primary:

```text
 slot_name | active | wal_status | retained
-----------+--------+------------+----------
 standby1  | f      | reserved   | 746 MB

 max_slot_wal_keep_size
------------------------
 -1
```

The inactive slot made the primary keep 746 MB of WAL in 10 seconds, in case the standby came back.
A standby that is gone for good — decommissioned without dropping its slot — makes `pg_wal` grow until
**the primary's disk fills up and it stops**. It also holds back VACUUM if the slot carries an `xmin`
(with `hot_standby_feedback`). Protections:

- `max_slot_wal_keep_size` (default `-1`, unlimited): cap retained WAL; a slot that exceeds it is
  invalidated (`wal_status = 'lost'`) and the standby must be rebuilt — better than an outage.
- Alert on inactive slots and on retained WAL size.
- Drop slots as part of decommissioning: `SELECT pg_drop_replication_slot('standby1');`

When the standby restarted it caught up from the retained WAL — the slot did its job:

```text
   state   |  lag
-----------+--------
 streaming | 733 MB
```

and the row committed while it was down was there:

```text
$ psql -p 54330 -d lab -Atc "select note from repl_demo where id = 3"
committed while standby is down
```

## Queries on the standby and conflicts

A standby replaying WAL may need to remove row versions that a long query on the standby is still
reading (because VACUUM on the primary removed them). PostgreSQL then either delays replay (up to
`max_standby_streaming_delay`, default 30 s) or cancels the standby query with "canceling statement due
to conflict with recovery". `hot_standby_feedback = on` makes the standby tell the primary which rows its
queries still need, avoiding most cancellations at the cost of some bloat on the primary. Choose per
standby: a reporting replica usually wants feedback on; a pure failover replica may not.

## Promotion

```bash
pg_ctl -D standby1 promote          # or SELECT pg_promote();
```

```text
LOG:  received promote request
LOG:  selected new timeline ID: 2
LOG:  archive recovery complete
```

```text
$ psql -p 54330 -d lab -c "insert into repl_demo values (4, 'written on the promoted standby') returning note"
              note
---------------------------------
 written on the promoted standby
```

The standby finished replaying the WAL it had received, became a read-write primary, and started a new
**timeline** (2), recorded in `pg_wal/00000002.history`. Timelines are how PostgreSQL tells the history
of the old and new primaries apart from the moment of the fork.

I promoted it while `pg_stat_replication` on the old primary still reported 526 MB of replay lag, then
compared the `pgbench_history` table on both servers: identical row counts and the same latest
timestamp. The lag was WAL received but not yet *replayed*; promotion replays everything already
received before opening for writes. WAL that has not been *received* is what an async failover loses.

The old primary is still running and accepting writes on timeline 1. Two primaries is **split brain**:
clients writing to both create diverging data that cannot be merged. Real failover must fence the old
primary (stop it, cut its network, or have the load balancer stop routing to it) *before* promoting.
Turning the old primary into a standby of the new one requires `pg_rewind` or a fresh base backup.
Lesson 9 covers the tooling that automates this.

## How It Actually Works

On the standby, the **startup process** applies WAL — the same code path as crash recovery, running
forever instead of stopping at the end of the log. A **walreceiver** process connects to the primary
using the replication protocol and asks to stream from a given LSN; on the primary a **walsender**
process reads WAL (from buffers or `pg_wal`) and sends it. The walreceiver writes and flushes the WAL
into the standby's `pg_wal` and reports its write, flush and replay positions back, which is what
`pg_stat_replication` and synchronous commit use.

A physical slot is a small record on the primary (in `pg_replslot/`) storing `restart_lsn` — the oldest
WAL the consumer still needs — and optionally an `xmin`. Checkpoints never remove WAL segments newer than
the oldest `restart_lsn` of any slot, which is both the guarantee and the disk-filling risk.

Because the standby is a byte-for-byte copy, both must run the same PostgreSQL major version on the
same platform and architecture; that is the difference from logical replication (next lesson).

## Common mistakes

- Replicas on the same host, rack or availability zone as the primary.
- Leaving slots for decommissioned standbys; no `max_slot_wal_keep_size`; no alert on retained WAL.
- A single synchronous standby with no fallback, so its failure stops writes.
- Promoting without fencing the old primary.
- Not monitoring lag, then discovering at failover time that the replica was hours behind.
- Assuming a cancelled synchronous commit was rolled back.

## Exercise

1. Build a primary and a standby on your machine with `pg_basebackup -R -S`. Verify with
   `pg_stat_replication` and `pg_is_in_recovery()`.
2. Run a long query on the standby while updating and vacuuming the same table on the primary, with
   `hot_standby_feedback` off and then on. Record what happens to the standby query each time.
3. Configure `synchronous_standby_names = 'ANY 1 (s1, s2)'` with two standbys (set `application_name` in
   each `primary_conninfo`). Stop one; show commits still succeed. Stop both; show what happens.
4. Set `max_slot_wal_keep_size = '200MB'`, stop the standby, generate WAL until the slot is invalidated,
   and record `wal_status` along the way.
5. Promote the standby, then rejoin the old primary as a standby of the new one using `pg_rewind`
   (it requires `wal_log_hints = on` or data checksums — which PostgreSQL 18 enables by default).
