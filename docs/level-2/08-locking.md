---
description: "PostgreSQL locking in practice: table lock modes, the ALTER TABLE queue, row locks, deadlocks, lock_timeout, SKIP LOCKED and advisory locks."
---

# 08 · Locks, Deadlocks & SKIP LOCKED

MVCC means readers and writers do not block each other. That does not mean PostgreSQL has no
locks — it has plenty — and the situations where they *do* block are behind some of the most
dramatic production incidents: a one-second `ALTER TABLE` that takes a whole site down, or a queue
of workers that all wait on the same row. This lesson reproduces those situations and the tools
that prevent them.

The demonstrations use several concurrent connections, so they are driven by a Python script
(psycopg 3.3) against PostgreSQL 18.6; the output shown is what it printed. You can reproduce each
one with two or three `psql` windows.

## Two families of locks

**Table-level (relation) locks** are taken automatically by every statement:

| Lock mode | Taken by | Conflicts with |
|-----------|----------|----------------|
| ACCESS SHARE | `SELECT` | ACCESS EXCLUSIVE only |
| ROW SHARE | `SELECT ... FOR UPDATE/SHARE` | EXCLUSIVE, ACCESS EXCLUSIVE |
| ROW EXCLUSIVE | `INSERT`, `UPDATE`, `DELETE`, `MERGE` | SHARE and stronger |
| SHARE UPDATE EXCLUSIVE | `VACUUM`, `ANALYZE`, `CREATE INDEX CONCURRENTLY`, some `ALTER TABLE` forms | itself and stronger |
| SHARE | `CREATE INDEX` (non-concurrent) | ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE and the exclusive modes (not other SHARE locks) — **blocks writes** |
| ACCESS EXCLUSIVE | most `ALTER TABLE`, `DROP`, `TRUNCATE`, `VACUUM FULL`, `REFRESH MATERIALIZED VIEW` | **everything, including SELECT** |

**Row-level locks** are taken by writes and by `SELECT ... FOR ...`:

| Clause | Taken implicitly by | Blocks |
|--------|---------------------|--------|
| `FOR UPDATE` | `DELETE`, `UPDATE` of key columns | all other row locks |
| `FOR NO KEY UPDATE` | ordinary `UPDATE` | FOR UPDATE, FOR NO KEY UPDATE, FOR SHARE |
| `FOR SHARE` | — | FOR UPDATE, FOR NO KEY UPDATE |
| `FOR KEY SHARE` | foreign-key checks | FOR UPDATE only |

The weaker `NO KEY UPDATE` and `KEY SHARE` modes exist so that inserting a child row (which takes
`KEY SHARE` on the parent to stop it being deleted) does not block ordinary updates of the parent's
non-key columns.

## 1. The ALTER TABLE lock queue

Session 1 has an open transaction that read `acct` (holding ACCESS SHARE until commit). Session 2
runs a quick `ALTER TABLE acct ADD COLUMN note text`, which needs ACCESS EXCLUSIVE and so waits.
Session 3 then runs a plain `SELECT`:

```text
  waiting: (47197, [47196], 'Lock', 'ALTER TABLE acct ADD COLUMN note text')
  waiting: (47198, [47197], 'Lock', 'SELECT count(*) FROM acct')
  locks on acct: [('AccessShareLock', True), ('AccessExclusiveLock', False), ('AccessShareLock', False)]
  the plain SELECT waited 1.3 s behind the ALTER
```

The `SELECT` is blocked — not by session 1, whose lock it is compatible with, but by the **waiting**
`ALTER TABLE` (`pg_blocking_pids` shows 47198 blocked by 47197). Lock requests queue in order, and a
new request that conflicts with one already waiting goes behind it. So a DDL statement that would
take a millisecond can freeze all traffic on a table for as long as the oldest transaction on it
stays open. On a busy table that is an outage within seconds.

The fix is to never let DDL wait long:

```sql
SET lock_timeout = '200ms';
ALTER TABLE acct ADD COLUMN note text;
```

```text
   55P03 canceling statement due to lock timeout
```

The DDL gives up, the queue drains, and your migration tool retries a moment later. Level 4 · 07
builds this into a zero-downtime migration procedure.

Diagnose blocking with:

```sql
SELECT pid, pg_blocking_pids(pid) AS blocked_by, wait_event_type, wait_event,
       now() - query_start AS waiting_for, left(query, 60) AS query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

## 2. Deadlocks

Two transfers in opposite directions. A locks account 1, B locks account 2, then each tries to lock
the other's:

```sql
-- A                                   -- B
UPDATE acct ... WHERE id = 1;          UPDATE acct ... WHERE id = 2;
UPDATE acct ... WHERE id = 2;  -- waits
                                       UPDATE acct ... WHERE id = 1;  -- waits → cycle
```

```text
A: 40P01 deadlock detected after 1.0 s
   Process 47225 waits for ShareLock on transaction 101437; blocked by process 47226.
   Process 47226 waits for ShareLock on transaction 101436; blocked by process 47225.
   HINT: See server log for query details.
B committed
```

PostgreSQL does not check for deadlocks on every wait — that would be expensive. A process that has
waited `deadlock_timeout` (1 s by default) runs the detector; if it finds a cycle, it aborts *itself*
with SQLSTATE `40P01`, and the other transaction proceeds. Here A started waiting first, so A's check
fired first and A was the victim.

Note the detail: each process waits for a **ShareLock on a transaction**. A row lock is recorded in
the row itself (its `xmax`, lesson 1); to wait for it, a backend waits on the lock that the holding
*transaction* holds on its own ID until it ends.

Prevent deadlocks by **acquiring locks in a consistent order** — for transfers, always lock the lower
account ID first:

```sql
SELECT id FROM acct WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- now update both rows in any order
```

and treat 40P01 like a serialization failure: roll back and retry (Level 1 · 07).

## 3. SKIP LOCKED: a job queue without a message broker

A naive queue worker does `SELECT ... WHERE NOT done ORDER BY id LIMIT 1 FOR UPDATE`. Every worker
picks the same first row; all but one wait. `SKIP LOCKED` makes each worker skip rows another
transaction has locked:

```sql
UPDATE jobs SET done = true
WHERE id = (SELECT id FROM jobs WHERE NOT done
            ORDER BY id
            FOR UPDATE SKIP LOCKED
            LIMIT 1)
RETURNING id;
```

```text
  worker1 got job 1 (transaction still open)
  worker2 got job 2
  worker2 next job 1
```

Worker 2 skipped job 1 because worker 1 held its lock. Then worker 1 "crashed" (rolled back) —
its lock vanished, job 1 was still `done = false`, and worker 2 picked it up. That rollback-on-crash
behaviour is what makes this pattern safe: as long as a job is processed and marked inside the same
transaction, a dead worker cannot lose a job. Level 3 · 09 builds a complete queue with retries
around it.

`SKIP LOCKED` intentionally gives an inconsistent view of the table, so use it only for
work-distribution queries like this.

## 4. NOWAIT and lock_timeout

```sql
SELECT * FROM acct WHERE id = 1 FOR UPDATE NOWAIT;
```

```text
   55P03 could not obtain lock on row in relation "acct"
```

`NOWAIT` fails immediately instead of waiting — useful in interactive flows ("someone else is
editing this record"). `lock_timeout` is the general form: a maximum wait for any lock, per
statement. Set it for migrations and admin sessions. `statement_timeout` limits total statement time,
and `idle_in_transaction_session_timeout` kills sessions that hold locks while doing nothing.

## 5. Advisory locks

Sometimes you need a lock on something that is not a row — "only one instance of the nightly report
job may run". Advisory locks are application-defined locks on a 64-bit number:

```text
  A try lock 42: True
  B try lock 42: False
  B try again  : True     (after A ran pg_advisory_unlock(42))
```

- `pg_advisory_lock(key)` waits; `pg_try_advisory_lock(key)` returns false immediately.
- Session-level locks (above) last until unlocked or disconnect. Transaction-level ones
  (`pg_advisory_xact_lock`) release automatically at commit or rollback — prefer them, because a
  session lock leaked by a pooled connection stays held for whoever uses that connection next.
- Derive keys from names with `hashtext('nightly-report')`, or use the two-argument form
  `(class, id)`.

## How It Actually Works

Table-level locks live in a shared-memory **lock table**: each entry records a lockable object (a
relation, a transaction ID, an advisory key, …), the modes granted, and a queue of waiters. A request
is granted only if it conflicts neither with granted modes nor with waiters ahead of it — the queue
fairness that produced the ALTER TABLE pile-up. To keep the common case cheap, weak relation locks
taken by ordinary DML (ACCESS SHARE, ROW SHARE, ROW EXCLUSIVE) use per-backend **fast-path** slots and
only move into the main table when someone requests a strong lock on the same relation.

Row locks are *not* in the lock table — there could be billions of them. A locker writes its
transaction ID into the row's `xmax` with flag bits saying "locked only" and the strength; when
several transactions share-lock a row, `xmax` points to a **MultiXact** ID listing them, stored in
`pg_multixact`. A waiter takes a short-lived tuple lock to queue fairly, then waits on the holder's
transaction ID lock. That is why every deadlock report mentions "ShareLock on transaction".

MultiXacts have their own 32-bit ID space and need freezing like transaction IDs; heavy use of
`FOR SHARE` / `FOR KEY SHARE` (including from foreign keys) on hot rows can make `pg_multixact`
grow and is worth monitoring with `mxid_age(datminmxid)`.

## Common mistakes

- Running `ALTER TABLE` on a hot table without `lock_timeout`.
- Plain `CREATE INDEX` (SHARE lock, blocks writes) on a busy table instead of
  `CREATE INDEX CONCURRENTLY`.
- Locking rows in inconsistent orders across code paths, then "fixing" deadlocks by raising
  `deadlock_timeout`.
- Queue workers without `SKIP LOCKED`, serialising on the first row.
- Session-level advisory locks with a connection pooler.
- Long transactions holding row locks while waiting on external calls.

## Exercise

1. Reproduce the ALTER TABLE queue with three `psql` sessions and observe it with the blocking query
   above. Then repeat with `SET lock_timeout = '1s'` in the DDL session.
2. Write two transfers that deadlock, then change them to lock accounts in ID order and show they
   no longer can.
3. Build a `jobs` table with 1,000 rows and run four worker processes that claim jobs with
   `SKIP LOCKED`, sleep 10 ms per job, and record which worker did which job. Verify every job was
   processed exactly once.
4. Use `pg_locks` to list the locks held by a session in the middle of an `UPDATE` transaction. Which
   relations appear, and in which modes?
