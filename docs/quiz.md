---
title: "Final Quiz"
description: "Test what you have learned across all four levels of PostgreSQL Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this return?\n```sql\ncreate table t(a int, b int);\ninsert into t values (1,null),(2,5);\n\nselect count(*), count(b), sum(b), avg(b) from t\n```\n",
    "options": [
      "1,1,5,5",
      "2,1,5,5.0000000000000000",
      "2,1,5,2.5",
      "2,2,5,2.5"
    ],
    "correctIndex": 1,
    "explanation": "Aggregates ignore NULLs; count(*) counts rows. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons)."
  },
  {
    "question": "What does this return?\n```sql\ncreate table t(a int);\ninsert into t values (1),(2),(3);\n\nselect a from t where a not in (select a from t where a > 5) order by a\n```\n",
    "options": [
      "(none)",
      "1; 2; 3",
      "Error",
      "1"
    ],
    "correctIndex": 1,
    "explanation": "An empty subquery makes NOT IN true (NULLs would make it unknown). Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons)."
  },
  {
    "question": "What does this return?\n```sql\ncreate table t(a int); insert into t values (1),(null);\n\nselect count(*) from t where a not in (select 1 union all select null)\n```\n",
    "options": [
      "1",
      "Error",
      "0",
      "2"
    ],
    "correctIndex": 2,
    "explanation": "NOT IN with a NULL in the list never returns true. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons)."
  },
  {
    "question": "What error appears?\n```sql\ncreate table t(a int);\ninsert into t values (1);\n\nselect a, count(*) from t\n```\n",
    "options": [
      "column \"t.a\" must appear in the GROUP BY clause or be used in an aggregate function",
      "syntax",
      "no error",
      "division by zero"
    ],
    "correctIndex": 0,
    "explanation": "Non-aggregated columns need GROUP BY. Verified by running it on PostgreSQL 18 (the error message is shown)."
  },
  {
    "question": "What does this return?\n```sql\ncreate table t(id int primary key, v int);\ninsert into t values (1,10);\n\ninsert into t values (1,99) on conflict (id) do update set v = excluded.v returning id, v\n```\n",
    "options": [
      "1,99",
      "Error",
      "(none)",
      "1,10"
    ],
    "correctIndex": 0,
    "explanation": "The upsert replaced the value. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons)."
  },
  {
    "question": "Which anomaly can Read Committed show inside one transaction?",
    "options": [
      "A non-repeatable read: the same query can return different results because each statement takes a new snapshot",
      "Dirty read",
      "None",
      "Lost commit"
    ],
    "correctIndex": 0,
    "explanation": "Use Repeatable Read or Serializable for consistent multi-statement reads."
  },
  {
    "question": "Why is a conditional UPDATE … WHERE not sold a safe way to claim a seat?",
    "options": [
      "The row lock makes it atomic: only one concurrent transaction changes the row and sees it returned",
      "It's slower",
      "It uses sleep",
      "Not safe"
    ],
    "correctIndex": 0,
    "explanation": "Let the database enforce invariants."
  },
  {
    "question": "What does MVCC mean for readers and writers?",
    "options": [
      "Readers lock",
      "Writers queue",
      "Readers don't block writers and writers don't block readers (they see different row versions)",
      "Everyone waits"
    ],
    "correctIndex": 2,
    "explanation": "Snapshots."
  },
  {
    "question": "Why does a table with many UPDATEs get bloated without vacuum?",
    "options": [
      "Locks",
      "Each update leaves a dead row version until vacuum reclaims it",
      "Indexes",
      "WAL"
    ],
    "correctIndex": 1,
    "explanation": "Tune autovacuum."
  },
  {
    "question": "What does EXPLAIN (ANALYZE, BUFFERS) tell you?",
    "options": [
      "Locks",
      "Estimates only",
      "Config",
      "The real plan with timings, row counts and buffer hits/reads"
    ],
    "correctIndex": 3,
    "explanation": "Compare estimates to actuals."
  },
  {
    "question": "A composite index on (a, b) is best for which query?",
    "options": [
      "WHERE a = ? AND b > ? (and WHERE a = ?)",
      "ORDER BY b only",
      "WHERE b = ? only",
      "LIKE '%x'"
    ],
    "correctIndex": 0,
    "explanation": "Leftmost prefix."
  },
  {
    "question": "Which index type fits jsonb containment (@>) queries?",
    "options": [
      "Hash",
      "BRIN",
      "GIN",
      "B-tree"
    ],
    "correctIndex": 2,
    "explanation": "Inverted index."
  },
  {
    "question": "How add an index to a busy table safely?",
    "options": [
      "CREATE INDEX",
      "LOCK TABLE",
      "REINDEX",
      "CREATE INDEX CONCURRENTLY"
    ],
    "correctIndex": 3,
    "explanation": "Avoids blocking writes."
  },
  {
    "question": "What does the planner rely on to estimate row counts?",
    "options": [
      "WAL",
      "Statistics gathered by ANALYZE",
      "Indexes",
      "Config only"
    ],
    "correctIndex": 1,
    "explanation": "Keep them fresh."
  },
  {
    "question": "A worker queue with many consumers should claim jobs with?",
    "options": [
      "LOCK TABLE",
      "SELECT … FOR UPDATE SKIP LOCKED",
      "ORDER BY random()",
      "Plain SELECT"
    ],
    "correctIndex": 1,
    "explanation": "No double processing, no waiting."
  },
  {
    "question": "What protects a multi-tenant table so one tenant can't read another's rows?",
    "options": [
      "Views only",
      "Naming",
      "Comments",
      "Row-level security policies on tenant_id (plus application checks)"
    ],
    "correctIndex": 3,
    "explanation": "Defence in depth."
  },
  {
    "question": "What do you need for point-in-time recovery?",
    "options": [
      "A base backup plus continuous WAL archiving",
      "Snapshots only",
      "Replica only",
      "pg_dump nightly"
    ],
    "correctIndex": 0,
    "explanation": "Practice restores."
  },
  {
    "question": "What does a streaming standby replay?",
    "options": [
      "Logs",
      "SQL",
      "WAL from the primary (physical, byte-identical copy)",
      "Rows"
    ],
    "correctIndex": 2,
    "explanation": "Great for HA and read scaling."
  },
  {
    "question": "Which setting avoids connection explosions?",
    "options": [
      "More RAM only",
      "A pooler (PgBouncer) with a modest max_connections",
      "Disable auth",
      "Huge max_connections"
    ],
    "correctIndex": 1,
    "explanation": "Process per connection."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Atomic seat booking and availability report

Finish the schema (a foreign key to the seat, a positive-price CHECK and a UNIQUE constraint so a seat can only be sold once), then claim seats with INSERT … ON CONFLICT DO NOTHING RETURNING so the second buyer of seat 2 gets no row, and produce the availability and revenue reports.

**Starter code**

```sql
CREATE TABLE seat (
  show_id int NOT NULL,
  seat_no int NOT NULL,
  price_cents int NOT NULL,            -- must be positive
  PRIMARY KEY (show_id, seat_no)
);
CREATE TABLE booking (
  id serial PRIMARY KEY,
  show_id int NOT NULL,
  seat_no int NOT NULL,
  customer text NOT NULL
  -- TODO: reference the seat and make sure a seat can be sold only once
);
INSERT INTO seat SELECT 1, g, 2500 FROM generate_series(1, 5) g;

-- TODO 1: three INSERTs claiming seats 2 (ann), 2 (bob) and 4 (bob). Use ON CONFLICT DO NOTHING RETURNING
--         id, seat_no, customer so that a lost race returns no row.
-- TODO 2: list every seat of show 1 with its customer or '(free)'.
-- TODO 3: one row with sold, free and revenue_cents (use FILTER).
```

**Expected output**

```text
id | seat_no | customer
----+---------+----------
  1 |       2 | ann
(1 row)

 id | seat_no | customer
----+---------+----------
(0 rows)

 id | seat_no | customer
----+---------+----------
  3 |       4 | bob
(1 row)

 seat_no | status
---------+--------
       1 | (free)
       2 | ann
       3 | (free)
       4 | bob
       5 | (free)
(5 rows)

 sold | free | revenue_cents
------+------+---------------
    2 |    3 |          5000
(1 row)
```

<details><summary>Show solution and explanation</summary>

<pre><code>CREATE TABLE seat (
  show_id int NOT NULL,
  seat_no int NOT NULL,
  price_cents int NOT NULL CHECK (price_cents &gt; 0),
  PRIMARY KEY (show_id, seat_no)
);
CREATE TABLE booking (
  id serial PRIMARY KEY,
  show_id int NOT NULL,
  seat_no int NOT NULL,
  customer text NOT NULL,
  FOREIGN KEY (show_id, seat_no) REFERENCES seat,
  UNIQUE (show_id, seat_no)                      -- a seat can be sold only once
);
INSERT INTO seat SELECT 1, g, 2500 FROM generate_series(1, 5) g;

-- Atomic claim: only one caller gets a row back for the same seat.
INSERT INTO booking (show_id, seat_no, customer) VALUES (1, 2, &#x27;ann&#x27;) ON CONFLICT DO NOTHING RETURNING id, seat_no, customer;
INSERT INTO booking (show_id, seat_no, customer) VALUES (1, 2, &#x27;bob&#x27;) ON CONFLICT DO NOTHING RETURNING id, seat_no, customer;
INSERT INTO booking (show_id, seat_no, customer) VALUES (1, 4, &#x27;bob&#x27;) ON CONFLICT DO NOTHING RETURNING id, seat_no, customer;

-- Availability report: free seats and revenue so far.
SELECT s.seat_no, COALESCE(b.customer, &#x27;(free)&#x27;) AS status
FROM seat s LEFT JOIN booking b USING (show_id, seat_no)
WHERE s.show_id = 1
ORDER BY s.seat_no;

SELECT count(b.id) AS sold,
       count(*) - count(b.id) AS free,
       sum(s.price_cents) FILTER (WHERE b.id IS NOT NULL) AS revenue_cents
FROM seat s LEFT JOIN booking b USING (show_id, seat_no)
WHERE s.show_id = 1;
</code></pre>

<p>The unique constraint makes the claim atomic, so a lost race simply returns no row. LEFT JOIN with FILTER computes sold, free and revenue in one pass. Verified by running the script on PostgreSQL 18 with psql.</p>
</details>

### Window functions, recursive CTE and JSON aggregation

Write four queries: rank reps per region with a percentage of the regional total, a running total per region, an indented org chart with a recursive CTE, and a per-region jsonb object of rep totals.

**Starter code**

```sql
CREATE TABLE sale (id serial PRIMARY KEY, region text, rep text, amount int, sold_on date);
INSERT INTO sale (region, rep, amount, sold_on) VALUES
 ('north','ann',500,'2026-01-05'), ('north','bob',300,'2026-01-20'), ('north','ann',700,'2026-02-03'),
 ('south','cy',400,'2026-01-11'), ('south','cy',900,'2026-02-14'), ('south','dee',200,'2026-02-20');
CREATE TABLE org (id int PRIMARY KEY, name text, boss int);
INSERT INTO org VALUES (1,'zoe',NULL),(2,'ann',1),(3,'bob',1),(4,'cy',2),(5,'dee',2);

-- 1. Rank reps within each region by total sales (rnk) and show each rep's percentage of the region total (pct, 1 decimal).
--    Columns: region, rep, total, rnk, pct  -- ordered by region, rnk.

-- 2. Running total of amount per region ordered by sold_on. Columns: region, sold_on, amount, running.

-- 3. Org chart with a recursive CTE starting from the person without a boss; indent each name by two spaces per depth.
--    Single column org_chart, ordered by depth then name.

-- 4. One row per region with a jsonb object of rep -> total (keys ordered by rep). Columns: region, by_rep.
```

**Expected output**

```text
region | rep | total | rnk | pct
--------+-----+-------+-----+------
 north  | ann |  1200 |   1 | 80.0
 north  | bob |   300 |   2 | 20.0
 south  | cy  |  1300 |   1 | 86.7
 south  | dee |   200 |   2 | 13.3
(4 rows)

 region |  sold_on   | amount | running
--------+------------+--------+---------
 north  | 2026-01-05 |    500 |     500
 north  | 2026-01-20 |    300 |     800
 north  | 2026-02-03 |    700 |    1500
 south  | 2026-01-11 |    400 |     400
 south  | 2026-02-14 |    900 |    1300
 south  | 2026-02-20 |    200 |    1500
(6 rows)

 org_chart
-----------
 zoe
   ann
   bob
     cy
     dee
(5 rows)

 region |          by_rep
--------+---------------------------
 north  | {"ann": 1200, "bob": 300}
 south  | {"cy": 1300, "dee": 200}
(2 rows)
```

<details><summary>Show solution and explanation</summary>

<pre><code>CREATE TABLE sale (id serial PRIMARY KEY, region text, rep text, amount int, sold_on date);
INSERT INTO sale (region, rep, amount, sold_on) VALUES
 (&#x27;north&#x27;,&#x27;ann&#x27;,500,&#x27;2026-01-05&#x27;), (&#x27;north&#x27;,&#x27;bob&#x27;,300,&#x27;2026-01-20&#x27;), (&#x27;north&#x27;,&#x27;ann&#x27;,700,&#x27;2026-02-03&#x27;),
 (&#x27;south&#x27;,&#x27;cy&#x27;,400,&#x27;2026-01-11&#x27;), (&#x27;south&#x27;,&#x27;cy&#x27;,900,&#x27;2026-02-14&#x27;), (&#x27;south&#x27;,&#x27;dee&#x27;,200,&#x27;2026-02-20&#x27;);
CREATE TABLE org (id int PRIMARY KEY, name text, boss int);
INSERT INTO org VALUES (1,&#x27;zoe&#x27;,NULL),(2,&#x27;ann&#x27;,1),(3,&#x27;bob&#x27;,1),(4,&#x27;cy&#x27;,2),(5,&#x27;dee&#x27;,2);

-- 1. Rank reps within each region by total sales, with each rep&#x27;s share of the region total.
WITH t AS (SELECT region, rep, sum(amount) AS total FROM sale GROUP BY region, rep)
SELECT region, rep, total,
       rank() OVER (PARTITION BY region ORDER BY total DESC) AS rnk,
       round(100.0 * total / sum(total) OVER (PARTITION BY region), 1) AS pct
FROM t ORDER BY region, rnk;

-- 2. Running total per region ordered by date.
SELECT region, sold_on, amount, sum(amount) OVER (PARTITION BY region ORDER BY sold_on) AS running
FROM sale ORDER BY region, sold_on;

-- 3. Everyone under zoe with their depth (recursive CTE).
WITH RECURSIVE tree AS (
  SELECT id, name, 0 AS depth FROM org WHERE boss IS NULL
  UNION ALL
  SELECT o.id, o.name, t.depth + 1 FROM org o JOIN tree t ON o.boss = t.id
)
SELECT repeat(&#x27;  &#x27;, depth) || name AS org_chart FROM tree ORDER BY depth, name;

-- 4. Aggregate as JSON.
SELECT region, jsonb_object_agg(rep, total ORDER BY rep) AS by_rep
FROM (SELECT region, rep, sum(amount) AS total FROM sale GROUP BY region, rep) x
GROUP BY region ORDER BY region;
</code></pre>

<p>Window functions compute per-group rankings and running totals without collapsing rows, a recursive CTE walks the hierarchy, and jsonb_object_agg builds JSON in the database. Verified by running the script on PostgreSQL 18 with psql.</p>
</details>
