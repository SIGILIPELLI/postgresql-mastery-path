---
description: "Level 1 project: design an event ticketing database where PostgreSQL itself prevents double-booked venues, double-sold seats and bad data, then back it up."
---

# 10 · Project — Event Ticketing Database

Ticketing is a good test of a schema because the hard rules are *concurrency* rules. Two
promoters must not book the same hall for overlapping evenings. Two fans clicking "buy" on seat
A-1-14 within the same millisecond must not both get it. A seat on hold must eventually be released.
Bugs here cost real money and very public embarrassment, and checking them in application code is
exactly the kind of check-then-act logic that fails under load.

In this project you will build a database in which **PostgreSQL itself enforces every one of those
rules**, using what Level 1 covered: types (lesson 3), schemas (4), roles and default privileges (5),
constraints (6), concurrency (7), PostgreSQL-specific SQL (8) and backups (9). Everything below was
run on PostgreSQL 18.6, and the outputs are the real ones.

## Requirements

1. Venues have seats arranged in sections and rows; each venue has a time zone.
2. Events take place at a venue during a time range of at most 12 hours. **A venue can host only one
   event at a time.**
3. A seat can be **held** (with an expiry) or **paid** at most once per event. Refunded or expired
   tickets free the seat again.
4. Customers are unique by email, ignoring case and surrounding spaces.
5. A waitlist where joining twice updates your request instead of erroring.
6. Three roles: an owner that runs migrations, an application role that can read and write but never
   delete tickets, and a read-only reporting role.
7. A tested backup.

## Step 1 — roles and database (as a superuser)

```sql
-- 01_roles.sql
CREATE ROLE ticketing_owner NOLOGIN;
CREATE ROLE ticketing_app    LOGIN PASSWORD 'change-me-app';
CREATE ROLE ticketing_report LOGIN PASSWORD 'change-me-report';
CREATE DATABASE ticketing OWNER ticketing_owner;
ALTER ROLE ticketing_app    IN DATABASE ticketing SET search_path = tix;
ALTER ROLE ticketing_report IN DATABASE ticketing SET search_path = tix;
```

`ticketing_owner` cannot log in: humans and deploy tools `SET ROLE` to it. The first time I put the
two `ALTER ROLE` lines in the schema script (run as the owner), it failed:

```text
ERROR:  permission denied to alter role
DETAIL:  Only roles with the CREATEROLE attribute and the ADMIN option on role "ticketing_app" may alter this role.
```

Changing *another role's* settings is a role-management privilege, not a database-ownership one —
so those lines belong with the superuser's setup.

## Step 2 — the schema (as the owner)

```sql
-- 02_schema.sql
\set ON_ERROR_STOP on
SET ROLE ticketing_owner;
CREATE SCHEMA tix;
CREATE EXTENSION IF NOT EXISTS btree_gist;  -- a "trusted" extension: the database owner may create it

CREATE TYPE tix.ticket_status AS ENUM ('held', 'paid', 'refunded', 'expired');

CREATE TABLE tix.venues (
  id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name      text NOT NULL UNIQUE,
  city      text NOT NULL,
  time_zone text NOT NULL CHECK (timestamptz '2000-01-01 00:00Z' AT TIME ZONE time_zone IS NOT NULL)  -- rejects unknown zones
);

CREATE TABLE tix.seats (
  id       bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  venue_id bigint NOT NULL REFERENCES tix.venues (id) ON DELETE CASCADE,
  section  text   NOT NULL,
  seat_row text   NOT NULL,
  number   int    NOT NULL CHECK (number > 0),
  UNIQUE (venue_id, section, seat_row, number)
);
CREATE INDEX ON tix.seats (venue_id);

CREATE TABLE tix.events (
  id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  public_id uuid   NOT NULL DEFAULT uuidv7() UNIQUE,
  venue_id  bigint NOT NULL REFERENCES tix.venues (id),
  title     text   NOT NULL CHECK (char_length(title) BETWEEN 3 AND 200),
  during    tstzrange NOT NULL CHECK (NOT isempty(during) AND upper(during) - lower(during) <= interval '12 hours'),
  base_price numeric(10,2) NOT NULL CHECK (base_price >= 0),
  CONSTRAINT no_double_booked_venue EXCLUDE USING gist (venue_id WITH =, during WITH &&)
);

CREATE TABLE tix.customers (
  id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email text NOT NULL,
  email_normalized text GENERATED ALWAYS AS (lower(trim(email))) STORED UNIQUE,
  name  text NOT NULL
);

CREATE TABLE tix.tickets (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  event_id    bigint NOT NULL REFERENCES tix.events (id),
  seat_id     bigint NOT NULL REFERENCES tix.seats (id),
  customer_id bigint NOT NULL REFERENCES tix.customers (id),
  price       numeric(10,2) NOT NULL CHECK (price >= 0),
  status      tix.ticket_status NOT NULL DEFAULT 'held',
  held_until  timestamptz,
  created_at  timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT held_needs_expiry CHECK (status <> 'held' OR held_until IS NOT NULL)
);
-- a seat can be held or sold once per event; refunded or expired tickets free it again
CREATE UNIQUE INDEX one_live_ticket_per_seat ON tix.tickets (event_id, seat_id) WHERE status IN ('held', 'paid');
CREATE INDEX ON tix.tickets (customer_id);
CREATE INDEX ON tix.tickets (seat_id);

CREATE TABLE tix.waitlist (
  event_id    bigint NOT NULL REFERENCES tix.events (id),
  customer_id bigint NOT NULL REFERENCES tix.customers (id),
  seats_wanted int NOT NULL CHECK (seats_wanted BETWEEN 1 AND 8),
  joined_at   timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (event_id, customer_id)
);
CREATE INDEX ON tix.waitlist (customer_id);

-- privileges: app reads/writes, report reads; future tables included
GRANT USAGE ON SCHEMA tix TO ticketing_app, ticketing_report;
GRANT SELECT ON ALL TABLES IN SCHEMA tix TO ticketing_report;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA tix TO ticketing_app;
GRANT DELETE ON tix.waitlist TO ticketing_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA tix GRANT SELECT ON TABLES TO ticketing_report;
ALTER DEFAULT PRIVILEGES IN SCHEMA tix GRANT SELECT, INSERT, UPDATE ON TABLES TO ticketing_app;
RESET ROLE;
```

Design decisions worth calling out:

- **`no_double_booked_venue`** is an exclusion constraint on `(venue_id =, during &&)` — requirement
  2 enforced by an index, race-free.
- **`one_live_ticket_per_seat`** is a *partial* unique index. Only `held` and `paid` tickets
  participate, so a refunded seat can be sold again without deleting history.
- **`held_needs_expiry`** makes a hold without an expiry impossible, so the cleanup job can always
  find stale holds.
- **`email_normalized`** is a stored generated column with a unique constraint: the app cannot
  forget to normalise.
- The time-zone check abuses `AT TIME ZONE`, which raises an error for an unknown zone. It uses a
  fixed timestamp, so the expression gives the same answer every time for a valid zone — CHECK
  expressions should never depend on `now()`.
- Every foreign-key column has an index (lesson 6).
- `public_id` is a `uuidv7()` for URLs, so ticket pages do not expose sequential IDs.

## Step 3 — seed data

```sql
-- 03_seed.sql
SET search_path = tix;
INSERT INTO venues (name, city, time_zone) VALUES
  ('Riverside Hall', 'Hyderabad', 'Asia/Kolkata'),
  ('Old Mill Studio', 'Pune', 'Asia/Kolkata');
-- Riverside: sections A and B, rows 1-10, 20 seats per row = 400 seats; Old Mill: 5 rows x 10
INSERT INTO seats (venue_id, section, seat_row, number)
SELECT 1, s, r::text, n FROM unnest(ARRAY['A','B']) s, generate_series(1,10) r, generate_series(1,20) n;
INSERT INTO seats (venue_id, section, seat_row, number)
SELECT 2, 'Main', r::text, n FROM generate_series(1,5) r, generate_series(1,10) n;
INSERT INTO events (venue_id, title, during, base_price) VALUES
  (1, 'Monsoon Jazz Night', '[2026-11-14 19:00+05:30, 2026-11-14 22:00+05:30)', 1200),
  (1, 'Classical Morning',  '[2026-11-15 09:00+05:30, 2026-11-15 11:30+05:30)', 800),
  (2, 'Improv Comedy',      '[2026-11-14 20:00+05:30, 2026-11-14 21:30+05:30)', 500);
INSERT INTO customers (email, name)
SELECT 'fan' || g || '@example.com', 'Fan ' || g FROM generate_series(1, 300) g;
-- 250 paid tickets for the jazz night
INSERT INTO tickets (event_id, seat_id, customer_id, price, status)
SELECT 1, s.id, s.id, 1200, 'paid' FROM seats s WHERE s.venue_id = 1 ORDER BY s.id LIMIT 250;
```

Run the three scripts:

```bash
psql -X -q -f 01_roles.sql
psql -X -q -d ticketing -f 02_schema.sql
psql -X -q -d ticketing -f 03_seed.sql
```

## Step 4 — prove the rules hold

Connect as the **application role** — testing as a superuser would hide permission problems — and
try to break every rule:

```text
$ psql -U ticketing_app -d ticketing -a -f 04_checks.sql
SELECT current_user, current_setting('search_path') AS path;
 current_user  | path
---------------+------
 ticketing_app | tix

-- 1. venue double-booked
INSERT INTO events (venue_id, title, during, base_price)
VALUES (1, 'Late Rock Show', '[2026-11-14 21:30+05:30, 2026-11-14 23:30+05:30)', 900);
ERROR:  conflicting key value violates exclusion constraint "no_double_booked_venue"
DETAIL:  Key (venue_id, during)=(1, ["2026-11-14 21:30:00+05:30","2026-11-14 23:30:00+05:30")) conflicts with existing key (venue_id, during)=(1, ["2026-11-14 19:00:00+05:30","2026-11-14 22:00:00+05:30")).

-- 2. seat sold twice
INSERT INTO tickets (event_id, seat_id, customer_id, price, status) VALUES (1, 1, 299, 1200, 'paid');
ERROR:  duplicate key value violates unique constraint "one_live_ticket_per_seat"
DETAIL:  Key (event_id, seat_id)=(1, 1) already exists.

-- 3. a hold without an expiry
INSERT INTO tickets (event_id, seat_id, customer_id, price) VALUES (1, 251, 299, 1200);
ERROR:  new row for relation "tickets" violates check constraint "held_needs_expiry"

-- 4. same email with different case/spaces
INSERT INTO customers (email, name) VALUES ('  FAN1@example.com', 'Impostor');
ERROR:  duplicate key value violates unique constraint "customers_email_normalized_key"
DETAIL:  Key (email_normalized)=(fan1@example.com) already exists.

-- 5. unknown time zone
INSERT INTO venues (name, city, time_zone) VALUES ('Nowhere', 'X', 'Mars/Olympus');
ERROR:  time zone "Mars/Olympus" not recognized

-- 6. the app may not delete tickets
DELETE FROM tickets WHERE id = 1;
ERROR:  permission denied for table tickets
```

Six rules, six rejections, zero application code.

## Step 5 — the everyday operations

```sql
-- hold a seat for 10 minutes
INSERT INTO tickets (event_id, seat_id, customer_id, price, held_until)
VALUES (1, 251, 299, 1200, now() + interval '10 minutes')
RETURNING id, status, held_until > now() AS hold_active;
```

```text
 id  | status | hold_active
-----+--------+-------------
 253 | held   | t
```

(IDs 251 and 252 were consumed by the two failed ticket inserts in step 4 — lesson 3's "gaps are
normal".)

```sql
-- refund: the seat becomes available again
UPDATE tickets SET status = 'refunded' WHERE event_id = 1 AND seat_id = 1
RETURNING id, old.status, new.status;
INSERT INTO tickets (event_id, seat_id, customer_id, price, status)
VALUES (1, 1, 300, 1200, 'paid') RETURNING id;
```

```text
 id | status |  status
----+--------+----------
  1 | paid   | refunded

 id
-----
 254
```

The waitlist is an upsert, so joining twice changes your request:

```sql
INSERT INTO waitlist (event_id, customer_id, seats_wanted) VALUES (1, 42, 4)
ON CONFLICT (event_id, customer_id) DO UPDATE SET seats_wanted = EXCLUDED.seats_wanted
RETURNING customer_id, seats_wanted;
```

```text
 customer_id | seats_wanted
-------------+--------------
          42 |            4
```

A sales dashboard, displaying each event's start in the venue's own time zone:

```sql
SELECT e.title,
       lower(e.during) AT TIME ZONE v.time_zone AS starts_local,
       count(s.id) AS capacity,
       count(t.id) FILTER (WHERE t.status = 'paid') AS sold,
       count(t.id) FILTER (WHERE t.status = 'held') AS held,
       coalesce(sum(t.price) FILTER (WHERE t.status = 'paid'), 0) AS revenue
FROM events e
JOIN venues v ON v.id = e.venue_id
JOIN seats s  ON s.venue_id = e.venue_id
LEFT JOIN tickets t ON t.event_id = e.id AND t.seat_id = s.id AND t.status IN ('paid','held')
GROUP BY e.id, e.title, e.during, v.time_zone
ORDER BY lower(e.during);
```

```text
       title        |    starts_local     | capacity | sold | held |  revenue
--------------------+---------------------+----------+------+------+-----------
 Monsoon Jazz Night | 2026-11-14 19:00:00 |      400 |  250 |    1 | 300000.00
 Improv Comedy      | 2026-11-14 20:00:00 |       50 |    0 |    0 |         0
 Classical Morning  | 2026-11-15 09:00:00 |      400 |    0 |    0 |         0
```

And the job that releases stale holds, meant to run every minute from a scheduler:

```sql
UPDATE tickets SET status = 'expired', held_until = NULL
WHERE status = 'held' AND held_until < now()
RETURNING id;
```

## Step 6 — a stampede test

The real question: what happens when 40 people try to buy the same 5 seats at the same instant?
This script opens 40 connections, lines them up behind a barrier, and releases them together. Each
picks one of the five seats at random:

```python
# rush.py — pip install "psycopg[binary]"
import random, threading, psycopg
from psycopg import errors

DSN = "host=localhost dbname=ticketing user=ticketing_app password=change-me-app"
seats = [r[0] for r in psycopg.connect(DSN).execute(
    "SELECT id FROM seats WHERE venue_id = 2 ORDER BY id LIMIT 5")]
results = {"sold": 0, "seat taken": 0}
lock = threading.Lock()
barrier = threading.Barrier(40)

def buyer(customer_id):
    with psycopg.connect(DSN, autocommit=True) as conn:
        seat = random.choice(seats)
        barrier.wait()                      # everyone fires at once
        try:
            conn.execute(
                "INSERT INTO tickets (event_id, seat_id, customer_id, price, status) "
                "VALUES (3, %s, %s, 500, 'paid')", (seat, customer_id))
            outcome = "sold"
        except errors.UniqueViolation:
            outcome = "seat taken"
        with lock:
            results[outcome] += 1

threads = [threading.Thread(target=buyer, args=(c,)) for c in range(1, 41)]
for t in threads: t.start()
for t in threads: t.join()
print(results)
print(psycopg.connect(DSN).execute(
    "SELECT count(*), count(DISTINCT seat_id) FROM tickets "
    "WHERE event_id = 3 AND status = 'paid'").fetchone())
```

```text
{'sold': 5, 'seat taken': 35}
(5, 5)
```

Five tickets, five distinct seats, 35 clean "seat taken" errors the application can turn into a
friendly message. No locks, no retries, no Redis — the partial unique index serialised the
conflicting inserts.

## Step 7 — back it up and prove the backup

```bash
pg_dump -d ticketing -Fc -f ticketing.dump
createdb ticketing_check
pg_restore -d ticketing_check -j 2 --exit-on-error ticketing.dump      # exit=0
```

Compare the two databases (tickets, seats, customers, indexes in `tix`):

```text
ticketing|257|450|300|16
ticketing_check|257|450|300|16
```

and confirm the exclusion constraint survived the restore:

```text
$ psql -d ticketing_check -Atc "select conname from pg_constraint where conname='no_double_booked_venue'"
no_double_booked_venue
```

Because the roles already exist in this cluster, ownership and grants restored too. For a new
cluster you would restore `pg_dumpall --globals-only` first (lesson 9).

## How It Actually Works

Each rule is enforced at a different layer, and it is worth knowing which:

- **CHECK constraints** (`held_needs_expiry`, prices, title length, time zone) run in the executor as
  each row is formed — before any index is touched. They are free in concurrency terms because they
  look at one row.
- **The exclusion constraint** is checked by its GiST index. Inserting an event searches the index
  for an entry with the same `venue_id` and an overlapping range. If it finds one from a transaction
  that has not finished, it waits for that transaction, then re-checks — the same mechanism a
  unique index uses.
- **The partial unique index** is a B-tree over `(event_id, seat_id)` that only contains rows whose
  status is `held` or `paid`. During the stampede, 40 backends tried to insert index entries for 5
  keys; the first inserter for each key won, and the others found a live conflicting entry and
  raised `23505`. When a ticket is refunded, the update produces a new row version whose status no
  longer matches the index predicate, so no entry exists for it and the seat key becomes free.
- **The `DELETE` refusal** is an ACL check before execution even starts.

## Extensions to try

1. Add `ticketing_report` dashboards as views in a `reporting` schema, owned by the owner role, and
   verify the report role can read them but not the underlying `tickets` table directly. (Hint:
   views run with the owner's privileges by default; look up `security_invoker`.)
2. Enforce "at most 8 paid or held tickets per customer per event". Can a constraint do it? If not,
   which isolation level makes a check-then-insert safe (lesson 7)?
3. Write `06_expire_holds.sql`, schedule it, and test it by inserting holds with `held_until` in the
   past.
4. Add a `price_tiers` table so seats in different sections cost different amounts, and make the
   ticket price default to the tier price.

## Exercise

Rebuild the project from scratch in your own cluster, then:

1. Run the stampede script with 200 buyers and 20 seats. Confirm sold equals seats.
2. Change the partial unique index to a plain `UNIQUE (event_id, seat_id)` and explain exactly which
   requirement breaks.
3. Try to book a second event at Riverside Hall from 22:00 to 23:00 on 14 November. Why does it
   succeed?
4. Restore your dump into a **new cluster** that has none of the roles, using
   `pg_dumpall --globals-only` first. Record every step you needed.
