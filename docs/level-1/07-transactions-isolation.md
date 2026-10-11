---
description: "PostgreSQL isolation levels in action: Read Committed, Repeatable Read, Serializable, lost updates, write skew, retries and savepoints."
---

# 07 · Transactions & Isolation Levels in Practice

Every statement in PostgreSQL runs inside a transaction — if you do not write `BEGIN`, each
statement is its own transaction. You already know the textbook ACID definitions; this lesson is
about the part that bites real applications: **what one transaction sees while others are changing
the same data**, and which anomalies each isolation level does and does not prevent.

Rather than describe the anomalies in the abstract, we reproduce them. Two sessions, A and B,
interleave their statements. You can follow along in two `psql` windows; to make the timing exact
and repeatable, the outputs below come from a small Python script (psycopg 3.3, PostgreSQL 18.6)
included at the end of the lesson.

Setup, in database `lab`:

```sql
CREATE TABLE accounts (id int PRIMARY KEY, balance int NOT NULL);
INSERT INTO accounts VALUES (1, 100);
CREATE TABLE doctors (name text PRIMARY KEY, on_call bool NOT NULL);
INSERT INTO doctors VALUES ('alice', true), ('bob', true);
```

## The three levels PostgreSQL actually has

You can request four standard levels, but `READ UNCOMMITTED` behaves exactly like `READ COMMITTED`:
PostgreSQL never shows uncommitted data to another transaction. So there are three:

| Level | Snapshot taken | Prevents | Still allows |
|-------|----------------|----------|--------------|
| Read Committed (default) | at the start of **each statement** | dirty reads | non-repeatable reads, lost updates in read-modify-write code, write skew |
| Repeatable Read | at the **first statement** of the transaction | the above + non-repeatable and phantom reads, lost updates (by raising an error) | write skew |
| Serializable | like Repeatable Read, plus dependency tracking | every anomaly: the result equals *some* serial order | — (but transactions can fail and must be retried) |

## 1. Read Committed: the ground moves between statements

| Step | Session A | Session B |
|------|-----------|-----------|
| 1 | `BEGIN; SELECT balance ...` → 100 | |
| 2 | | `UPDATE accounts SET balance = 50 ...; COMMIT;` |
| 3 | `SELECT balance ...` → **50** | |

```text
A reads: 100
B committed balance=50; A reads again in the SAME transaction: 50
```

Same transaction, same query, different answer. For most web requests that is fine. For a report
that runs ten queries and expects them to agree with each other (a total and its breakdown), it is a
bug: the numbers will not add up if data changes mid-report.

## 2. Repeatable Read: a frozen snapshot

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
```

```text
A reads: 100
B committed balance=50; A reads again: 100
A's update: SerializationFailure 40001 could not serialize access due to concurrent update
```

A keeps seeing 100 for the rest of its transaction — perfect for consistent reports and for
`pg_dump`, which uses exactly this to produce a consistent backup.

But then A tries to *update* the row B changed. A's snapshot says the balance is 100; the real,
committed balance is 50. Rather than silently overwrite B's work based on stale data, PostgreSQL
raises SQLSTATE **40001**. The application must roll back and **retry the whole transaction**.

## 3. The lost update in Read Committed

The classic bug: read a value into application code, compute, write it back.

```python
balance = select("SELECT balance FROM accounts WHERE id = 1")   # both read 100
update("UPDATE accounts SET balance = %s WHERE id = 1", balance - amount)
```

Two withdrawals, 30 and 20, run concurrently:

```text
two withdrawals of 30 and 20 from 100 leave: 80
```

It should be 50. B's write was based on the 100 it read before A committed, so A's withdrawal
vanished. Nothing errored. Three fixes, in order of preference:

1. **Let the database do the arithmetic:** `UPDATE accounts SET balance = balance - 20 WHERE id = 1`.
   The second update waits for the first transaction's row lock, then re-reads the *latest* row
   version before applying its change:

    ```text
    B is blocked waiting for A's row lock: True
    result: 50
    ```

2. **Lock what you read:** `SELECT balance FROM accounts WHERE id = 1 FOR UPDATE` — the second
   reader waits until the first transaction finishes (Level 2 · 08 covers row locks).
3. **Use Repeatable Read** and retry on 40001.

Add a `CHECK (balance >= 0)` and the database will also refuse to overdraw, no matter how the
withdrawals interleave.

## 4. Write skew: the anomaly Repeatable Read misses

Rule: at least one doctor must stay on call. Alice and Bob both feel unwell at the same moment:

| Step | Session A (Alice) | Session B (Bob) |
|------|-------------------|-----------------|
| 1 | count on-call doctors → 2, "fine, I can leave" | count on-call doctors → 2, "fine, I can leave" |
| 2 | `UPDATE doctors SET on_call = false WHERE name = 'alice'` | `UPDATE doctors SET on_call = false WHERE name = 'bob'` |
| 3 | `COMMIT` | `COMMIT` |

```text
[REPEATABLE READ] A sees 2 on call, B sees 2 on call
[REPEATABLE READ] both committed
[REPEATABLE READ] on call now: 0
```

Nobody is on call. The two transactions updated **different** rows, so there was no write conflict
to detect; each was consistent with its own snapshot, but the combination violates the rule. Under
Serializable:

```text
[SERIALIZABLE] A sees 2 on call, B sees 2 on call
[SERIALIZABLE] B commit failed: 40001 could not serialize access due to read/write dependencies among transactions
[SERIALIZABLE] on call now: 1
```

PostgreSQL noticed that each transaction read data the other one wrote — a cycle that no serial
order could produce — and aborted one of them. Retried, B would see only one doctor on call and
stay.

## Choosing a level

- **Read Committed** for most OLTP work, combined with atomic `UPDATE`s, constraints, and
  `SELECT ... FOR UPDATE` where you read-then-write.
- **Repeatable Read** for multi-query reports and exports that must be internally consistent.
- **Serializable** when business rules span multiple rows and you would rather not reason about
  every interleaving. The price is retries and some overhead for tracking reads.

Whatever you choose above Read Committed, your code **must** retry on SQLSTATE `40001` (and
`40P01`, deadlock detected). A minimal retry loop:

```python
import psycopg
from psycopg import errors

def transfer(conn, src, dst, amount, attempts=5):
    for attempt in range(attempts):
        try:
            with conn.transaction():
                conn.execute("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE")
                conn.execute("UPDATE accounts SET balance = balance - %s WHERE id = %s", (amount, src))
                conn.execute("UPDATE accounts SET balance = balance + %s WHERE id = %s", (amount, dst))
            return
        except (errors.SerializationFailure, errors.DeadlockDetected):
            if attempt == attempts - 1:
                raise
```

Keep transactions short, and never put anything with side effects outside the database (sending an
email, charging a card) inside a block that might be retried.

## 5. Errors abort the transaction; savepoints contain them

```text
dup: 23505
next statement: 25P02 current transaction is aborted, commands ignored until end of transaction block
```

After any error inside a transaction block, PostgreSQL refuses every further statement until you
`ROLLBACK`. You cannot "catch and continue" — unless you set a **savepoint** first:

```sql
BEGIN;
INSERT INTO accounts VALUES (2, 10);
SAVEPOINT before_risky;
INSERT INTO accounts VALUES (2, 10);           -- fails: duplicate key
ROLLBACK TO SAVEPOINT before_risky;            -- undo just that part
INSERT INTO accounts VALUES (3, 30);
COMMIT;
```

```text
dup inside savepoint: 23505
rows after commit: [1, 2, 3]
```

This is what psql's `ON_ERROR_ROLLBACK` (lesson 2) does automatically, and what ORMs use for
"nested transactions". Savepoints are not free — each one is a subtransaction with its own ID, and
thousands of them in one transaction can hurt performance — so use them around genuinely risky
steps, not every statement.

## How It Actually Works

Isolation in PostgreSQL is built on **snapshots**. A snapshot records which transactions had
committed at the moment it was taken: everything committed before it is visible, everything still
running or started later is invisible. Each row version carries the ID of the transaction that
created it (`xmin`) and the one that deleted or replaced it (`xmax`), and visibility is decided by
comparing those IDs against your snapshot. Level 2 · 01 takes this apart with real `xmin`/`xmax`
values.

- In **Read Committed**, a new snapshot is taken for every statement. When an `UPDATE` finds that
  its target row was changed by a transaction that committed after the snapshot, it waits for that
  transaction's row lock, then re-evaluates its `WHERE` clause against the *newest* row version and
  updates that ("EvalPlanQual"). That is why `balance = balance - 20` is safe.
- In **Repeatable Read**, there is one snapshot for the whole transaction. Re-reading the newest
  version would break the snapshot, so the same situation raises 40001 instead.
- **Serializable** (Serializable Snapshot Isolation, SSI) adds "SIREAD" predicate locks that record
  what each transaction *read*. They never block anybody. At commit time PostgreSQL looks for a
  dangerous pattern of read/write dependencies between concurrent transactions and aborts one
  participant. It can produce false positives — aborting transactions that would actually have been
  fine — which is another reason the retry loop is mandatory.

## Common mistakes

- Read-modify-write in application code under Read Committed (the lost update).
- Using Repeatable Read or Serializable without a retry loop; 40001 errors then surface as random
  500s under load.
- Long transactions — a forgotten `BEGIN` in a psql window, a request that waits on an HTTP call —
  which hold locks and, as Level 2 shows, block VACUUM cleanup cluster-wide.
- Assuming `READ UNCOMMITTED` lets you peek at uncommitted data. It does not.
- Catching an exception inside a transaction and continuing without a savepoint.

## Exercise

1. Reproduce the Read Committed and Repeatable Read experiments in two `psql` windows. In the
   Repeatable Read case, what happens if B *has not yet committed* when A issues its update?
2. Reproduce the lost update, then fix it three ways: atomic update, `SELECT ... FOR UPDATE`, and
   Repeatable Read with a retry loop.
3. Model a different write-skew case — for example two users booking the last two seats where the
   rule is "at most N seats" — and show that Serializable prevents it.
4. Write a retry helper in your language of choice that retries on 40001 and 40P01 with a small
   random back-off, and test it by running two conflicting transactions concurrently.

## Appendix: the demonstration script

```python
# iso.py — run with: pip install "psycopg[binary]"; python iso.py
import threading
import psycopg

DSN = "host=localhost port=5432 user=postgres dbname=lab"
conn = lambda: psycopg.connect(DSN)
setup = conn(); setup.autocommit = True
bal = lambda c: c.execute("SELECT balance FROM accounts WHERE id = 1").fetchone()[0]
def reset():
    setup.execute("UPDATE accounts SET balance = 100")
    setup.execute("UPDATE doctors SET on_call = true")

# 1. Read Committed
a, b = conn(), conn()
print("A reads:", bal(a))
b.execute("UPDATE accounts SET balance = 50 WHERE id = 1"); b.commit()
print("A reads again:", bal(a)); a.commit(); reset()

# 2. Repeatable Read
a, b = conn(), conn()
a.execute("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
print("A reads:", bal(a))
b.execute("UPDATE accounts SET balance = 50 WHERE id = 1"); b.commit()
print("A reads again:", bal(a))
try:
    a.execute("UPDATE accounts SET balance = balance - 10 WHERE id = 1")
except psycopg.Error as e:
    print("A's update:", e.sqlstate, e)
a.rollback(); reset()

# 3. Lost update, then the atomic fix
a, b = conn(), conn()
va, vb = bal(a), bal(b)
a.execute("UPDATE accounts SET balance = %s WHERE id = 1", (va - 30,)); a.commit()
b.execute("UPDATE accounts SET balance = %s WHERE id = 1", (vb - 20,)); b.commit()
print("lost update leaves:", bal(setup)); reset()

a, b = conn(), conn()
a.execute("UPDATE accounts SET balance = balance - 30 WHERE id = 1")
t = threading.Thread(target=lambda: (b.execute(
    "UPDATE accounts SET balance = balance - 20 WHERE id = 1"), b.commit()))
t.start(); t.join(0.5); print("B blocked:", t.is_alive())
a.commit(); t.join(); print("result:", bal(setup)); reset()

# 4. Write skew
for level in ("REPEATABLE READ", "SERIALIZABLE"):
    a, b = conn(), conn()
    for c in (a, b):
        c.execute(f"SET TRANSACTION ISOLATION LEVEL {level}")
        c.execute("SELECT count(*) FROM doctors WHERE on_call")
    a.execute("UPDATE doctors SET on_call = false WHERE name = 'alice'")
    b.execute("UPDATE doctors SET on_call = false WHERE name = 'bob'")
    a.commit()
    try:
        b.commit(); print(level, "both committed")
    except psycopg.Error as e:
        print(level, "B failed:", e.sqlstate)
    reset()
```
