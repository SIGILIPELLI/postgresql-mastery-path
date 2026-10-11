---
description: "A reliable job queue inside PostgreSQL: LISTEN/NOTIFY, SKIP LOCKED claims, retries with back-off, dead jobs and crash safety."
---

# 09 · LISTEN/NOTIFY & Job Queues in Postgres

Many applications add a message broker the moment they need background jobs: send this email, resize
that image, call this webhook. For a large range of workloads PostgreSQL can do the job itself, with a
property no external broker gives you for free: **the job is enqueued in the same transaction as the
data change that caused it**. If the order insert rolls back, the "send confirmation" job never
existed. No dual writes, no outbox reconciliation.

This lesson builds a working queue from three PostgreSQL features — `LISTEN/NOTIFY`, `SKIP LOCKED`
(Level 2 · 08) and transactions — runs it with four workers, and is honest about where it stops
scaling. Driven by Python (psycopg 3.3) on PostgreSQL 18.6.

## LISTEN and NOTIFY

`NOTIFY channel, 'payload'` (or `SELECT pg_notify('channel', 'payload')`) sends a message to every
session that has run `LISTEN channel`. Two properties matter:

```text
before commit, notifications: []
after commit, notifications: ['email']
three notifies in one transaction: ['x', 'y']
```

1. **Notifications are transactional**: they are delivered only when the sending transaction commits,
   and discarded if it rolls back. A listener never hears about a job that does not exist.
2. **Identical notifications in one transaction are collapsed**: `pg_notify('jobs','x')` twice plus
   `('jobs','y')` delivered `x` and `y` once each.

And two limits:

- They are **not stored**. A listener that is disconnected when the notification fires never gets it.
  So NOTIFY is a *wake-up signal*, never the job itself. The job lives in a table.
- Payloads are limited to under 8,000 bytes by default, and the server-wide notification queue is
  bounded (8 GB). A listener that never reads its notifications eventually blocks the queue for
  everyone — a reason to keep listener connections dedicated and attentive.

## The job table

```sql
CREATE TABLE job_queue (
  id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  kind         text NOT NULL,
  payload      jsonb NOT NULL DEFAULT '{}',
  run_at       timestamptz NOT NULL DEFAULT now(),
  attempts     int NOT NULL DEFAULT 0,
  max_attempts int NOT NULL DEFAULT 3,
  last_error   text,
  done_at      timestamptz
);
CREATE INDEX job_queue_ready ON job_queue (run_at) WHERE done_at IS NULL;

CREATE FUNCTION job_queue_notify() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  PERFORM pg_notify('jobs', NEW.kind);
  RETURN NULL;
END $$;
CREATE TRIGGER job_queue_notify AFTER INSERT ON job_queue
  FOR EACH ROW EXECUTE FUNCTION job_queue_notify();
```

- `run_at` lets jobs be scheduled in the future, and is how retries are delayed.
- The **partial index** contains only unfinished jobs, so it stays small no matter how much history
  accumulates, and the claim query reads it in `run_at` order.
- The trigger wakes listeners when a job is inserted — by any code path, including plain SQL.

Enqueuing is an ordinary insert, inside your business transaction:

```sql
BEGIN;
INSERT INTO orders (...) VALUES (...);
INSERT INTO job_queue (kind, payload) VALUES ('send_receipt', '{"order_id": 1234}');
COMMIT;
```

## The worker

```python
# worker.py — a Postgres-backed job queue: SKIP LOCKED claims, retries with back-off, LISTEN wake-ups
import collections, threading, time, psycopg

DSN = "host=localhost dbname=adv user=postgres"
CLAIM = """
SELECT id, kind, payload, attempts FROM job_queue
WHERE done_at IS NULL AND run_at <= now() AND attempts < max_attempts
ORDER BY run_at
FOR UPDATE SKIP LOCKED
LIMIT 1"""

def handle(kind, payload, attempts):
    time.sleep(0.005)                              # pretend to work
    if kind == "flaky" and attempts == 0:
        raise RuntimeError("upstream timeout")      # fails on the first try only
    if kind == "broken":
        raise ValueError("bad payload")             # always fails

def worker(name, stop, stats):
    conn = psycopg.connect(DSN)
    listen = psycopg.connect(DSN, autocommit=True)
    listen.execute("LISTEN jobs")
    while not stop.is_set():
        with conn.transaction():
            job = conn.execute(CLAIM).fetchone()
            if job:
                job_id, kind, payload, attempts = job
                try:
                    handle(kind, payload, attempts)
                    conn.execute("UPDATE job_queue SET done_at = now(), attempts = attempts + 1 "
                                 "WHERE id = %s", (job_id,))
                    stats[name] += 1
                except Exception as e:
                    conn.execute("""UPDATE job_queue SET attempts = attempts + 1, last_error = %s,
                                    run_at = now() + make_interval(secs => 0.1 * power(2, attempts))
                                    WHERE id = %s""", (repr(e), job_id))
        if not job:
            for _ in listen.notifies(timeout=0.2, stop_after=1):   # sleep until notified (or 200 ms)
                pass
```

The design:

- **Claim and finish in one transaction.** The row lock from `FOR UPDATE` is held while the job runs,
  and released by the same commit that marks it done. `SKIP LOCKED` makes other workers pass over it.
- **Failures are recorded, not raised.** The error is stored, `attempts` incremented, and `run_at`
  pushed into the future with exponential back-off (0.1 s, 0.2 s, 0.4 s … — use seconds to minutes in
  production). After `max_attempts` the job is left as a "dead" job for a human to inspect.
- **Idle workers block on LISTEN** instead of polling hard, but still wake every 200 ms so that
  delayed retries (whose `run_at` arrives without any new insert) get picked up. Notifications are an
  optimisation; correctness never depends on them.

## Running it

Four worker threads, then 400 jobs inserted in one statement: every 10th is `flaky` (fails once), job 7
is `broken` (always fails), the rest succeed.

```text
drained in 1.14 s; jobs completed per worker: {'w0': 96, 'w1': 98, 'w2': 101, 'w3': 104}
('broken', 0, 1, 3)
('email', 359, 0, 1)
('flaky', 40, 0, 2)
[(7, 3, "ValueError('bad payload')")]
```

(Columns: kind, done, dead, max attempts used.) Work was spread evenly across workers; all 40 flaky
jobs succeeded on their second attempt after back-off; the broken job was tried three times and parked
with its error message; 399 of 400 completed and none was processed twice — each `done_at` is set
exactly once, under the lock.

## What happens when a worker dies

A worker claims a job and then its process is killed:

```text
worker claimed job 401
another worker sees it as claimable: []
after the worker died: [(401, 0, None)]
```

While the worker lived, the job was locked and invisible to others. When its connection ended, the
transaction rolled back, the lock vanished, and the job was claimable again with `attempts` still 0 —
no heartbeat, no visibility timeout, no reconciliation job. That is the payoff of tying the job's state
to a database transaction.

The flip side: because the transaction stays open for the job's duration, **long jobs hold a
transaction open**, which holds back VACUUM for the whole database (Level 2 · 02) and occupies a
connection. For jobs that take minutes, switch to a *lease* model: claim by setting
`locked_until = now() + interval '5 minutes'` and commit immediately, do the work outside any
transaction, then mark it done; a crashed worker's lease simply expires.

## Keeping the queue healthy

- **Bloat.** Every job is inserted, updated at least once, and eventually deleted — a high-churn table.
  Delete or archive completed jobs regularly (or partition by day and drop partitions, lesson 6), and
  give the table aggressive autovacuum settings.
- **Monitoring.** Alert on queue depth (`count(*) WHERE done_at IS NULL AND run_at <= now()`), on the
  age of the oldest ready job, and on dead jobs.
- **Idempotency.** A job can run twice if a worker finishes the side effect (sends the email) and then
  crashes before committing. Design handlers to be safe to repeat, e.g. with an idempotency key.
- **Ordering.** `ORDER BY run_at` with `SKIP LOCKED` is roughly FIFO, not strictly ordered. If jobs for
  the same entity must run in order, claim per entity (for example with advisory locks on the entity
  ID).

## When to use something else

A PostgreSQL queue is the right default when jobs relate to data in the same database and volumes are
moderate; its ceiling depends on hardware, job duration and batch size, so measure it for your workload
(exercise 2 below) rather than trusting anyone's rule of thumb. Reach for a dedicated broker when you
need very high throughput (tens of thousands of messages per second sustained), fan-out to many
independent consumers with replay (the territory of log-based brokers such as Apache Kafka), or
cross-service messaging where the database is not the source of truth. Mature libraries implement this
exact pattern (for example River for Go, Oban for Elixir, Graphile Worker for Node, Solid Queue for
Rails); prefer one over hand-rolling for production.

## How It Actually Works

`NOTIFY` writes the message into the transaction's pending list; at commit, pending notifications are
appended to a shared queue (stored in `pg_notify/` SLRU files), and listening backends are signalled.
Each listener reads new entries the next time it is idle or between transactions, then the client
library receives them as asynchronous messages on the connection. Delivery order follows commit order,
which is why a listener never sees a notification before the corresponding row is visible.

`FOR UPDATE SKIP LOCKED` tries to lock each candidate row; if the row's `xmax` shows a lock held by a
running transaction, it skips the row instead of waiting. Combined with `LIMIT 1` and a partial index
in `run_at` order, each claim touches only a few index entries, so workers do not contend with each
other. The lock lives in the tuple (Level 2 · 08), so when a backend dies its transaction aborts and
its locks evaporate — the crash safety shown above requires no extra code.

## Common mistakes

- Putting the job data in the NOTIFY payload and losing it when no one is listening.
- Polling in a tight loop without `SKIP LOCKED`, so all workers fight over the same row.
- Holding a transaction open for jobs that take minutes.
- Never deleting completed jobs.
- Non-idempotent job handlers.
- Retrying forever with no back-off and no dead-job state.

## Exercise

1. Add a `priority` column and change the claim query and index so high-priority jobs run first,
   while still using a partial index.
2. Convert the worker to claim up to 10 jobs at once (`LIMIT 10`) and measure throughput with 4
   workers and 10,000 jobs.
3. Implement the lease model for long jobs (`locked_until`), kill a worker mid-job, and show the job is
   retried after the lease expires.
4. Add a nightly maintenance step that moves completed jobs older than 7 days into an archive table in
   batches, using a procedure that commits between batches (lesson 3).
