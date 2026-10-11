---
description: "Level 3 project: a multi-tenant task tracker with RLS, full-text search, labels, audit triggers and a partitioned audit log, fully tested."
---

# 10 · Project — Multi-Tenant SaaS Backend

This project puts Level 3 together in the schema of a small task-tracking SaaS product. Several
companies (tenants) share one database. Each must see only its own projects, tasks and history, even
if the application has a bug. Users search tasks by text and filter by labels. Every change is audited
with the real end user's identity. The audit log is partitioned by month so old history can be
dropped cheaply.

The features, and the lesson each comes from:

| Need | Feature |
|------|---------|
| tenant isolation that survives application bugs | row-level security (lesson 7) |
| search box | stored, weighted `tsvector` + GIN (lesson 2) |
| label filters | `text[]` + GIN (Level 2 · 05) |
| `updated_at` and audit trail | triggers (lesson 4) |
| who changed it | per-transaction settings (lessons 4 and 7) |
| cheap retention of history | range partitioning (lesson 6) |
| search API | SQL function (lesson 3) |

Everything below was run on PostgreSQL 18.6; the test output is real, including two errors I hit
while building it.

## Step 1 — roles, schema and policies

The table owner never logs in, the application connects as `tasks_app`, and every tenant table has
`FORCE ROW LEVEL SECURITY` so not even the owner bypasses the policies.

```sql
-- as superuser
CREATE ROLE tasks_owner NOLOGIN;
CREATE ROLE tasks_app LOGIN PASSWORD 'tasks-app-dev';
CREATE DATABASE tasks OWNER tasks_owner;
\c tasks
CREATE EXTENSION IF NOT EXISTS pg_trgm;
SET ROLE tasks_owner;
CREATE SCHEMA app;
GRANT USAGE ON SCHEMA app TO tasks_app;

CREATE FUNCTION app.current_tenant() RETURNS bigint LANGUAGE sql STABLE
RETURN nullif(current_setting('app.tenant_id', true), '')::bigint;

CREATE TABLE app.tenants (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name text NOT NULL UNIQUE,
  plan text NOT NULL CHECK (plan IN ('free', 'pro'))
);
CREATE TABLE app.projects (
  tenant_id bigint NOT NULL REFERENCES app.tenants,
  id bigint GENERATED ALWAYS AS IDENTITY,
  name text NOT NULL,
  PRIMARY KEY (tenant_id, id),
  UNIQUE (tenant_id, name)
);
CREATE TABLE app.tasks (
  tenant_id bigint NOT NULL,
  id bigint GENERATED ALWAYS AS IDENTITY,
  project_id bigint NOT NULL,
  title text NOT NULL,
  body text NOT NULL DEFAULT '',
  status text NOT NULL DEFAULT 'todo' CHECK (status IN ('todo', 'doing', 'done')),
  labels text[] NOT NULL DEFAULT '{}',
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  search tsvector GENERATED ALWAYS AS (
    setweight(to_tsvector('english', title), 'A') || setweight(to_tsvector('english', body), 'B')) STORED,
  PRIMARY KEY (tenant_id, id),
  FOREIGN KEY (tenant_id, project_id) REFERENCES app.projects (tenant_id, id)
);
CREATE INDEX tasks_project_idx ON app.tasks (tenant_id, project_id, status);
CREATE INDEX tasks_search_idx  ON app.tasks USING gin (search);
CREATE INDEX tasks_labels_idx  ON app.tasks USING gin (labels);

-- audit log, partitioned by month
CREATE TABLE app.audit_log (
  tenant_id bigint,
  at timestamptz NOT NULL DEFAULT clock_timestamp(),
  table_name text NOT NULL,
  op text NOT NULL,
  row_id bigint,
  changed jsonb,
  actor text DEFAULT current_setting('app.actor', true)
) PARTITION BY RANGE (at);
CREATE TABLE app.audit_log_2026_10 PARTITION OF app.audit_log FOR VALUES FROM ('2026-10-01') TO ('2026-11-01');
CREATE TABLE app.audit_log_2026_11 PARTITION OF app.audit_log FOR VALUES FROM ('2026-11-01') TO ('2026-12-01');
CREATE INDEX ON app.audit_log (tenant_id, at);

CREATE FUNCTION app.touch() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN NEW.updated_at := now(); RETURN NEW; END $$;
CREATE TRIGGER tasks_touch BEFORE UPDATE ON app.tasks FOR EACH ROW
  WHEN ((OLD.project_id, OLD.title, OLD.body, OLD.status, OLD.labels)
        IS DISTINCT FROM (NEW.project_id, NEW.title, NEW.body, NEW.status, NEW.labels))
  EXECUTE FUNCTION app.touch();

CREATE FUNCTION app.audit() RETURNS trigger LANGUAGE plpgsql SECURITY DEFINER
SET search_path = pg_catalog, pg_temp AS $$
DECLARE o jsonb; n jsonb; diff jsonb;
BEGIN
  IF TG_OP <> 'INSERT' THEN o := to_jsonb(OLD) - 'search'; END IF;
  IF TG_OP <> 'DELETE' THEN n := to_jsonb(NEW) - 'search'; END IF;
  IF TG_OP = 'UPDATE' THEN
    SELECT jsonb_object_agg(key, value) INTO diff FROM jsonb_each(n) WHERE o->key IS DISTINCT FROM value;
    IF diff IS NULL THEN RETURN NULL; END IF;
  ELSE
    diff := coalesce(n, o);
  END IF;
  INSERT INTO app.audit_log (tenant_id, table_name, op, row_id, changed)
  VALUES ((coalesce(n, o)->>'tenant_id')::bigint, TG_TABLE_NAME, TG_OP, (coalesce(n, o)->>'id')::bigint, diff);
  RETURN NULL;
END $$;
CREATE TRIGGER tasks_audit AFTER INSERT OR UPDATE OR DELETE ON app.tasks FOR EACH ROW EXECUTE FUNCTION app.audit();
CREATE TRIGGER projects_audit AFTER INSERT OR UPDATE OR DELETE ON app.projects FOR EACH ROW EXECUTE FUNCTION app.audit();

-- row-level security on every tenant table
ALTER TABLE app.projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE app.projects FORCE ROW LEVEL SECURITY;
ALTER TABLE app.tasks ENABLE ROW LEVEL SECURITY;
ALTER TABLE app.tasks FORCE ROW LEVEL SECURITY;
ALTER TABLE app.audit_log ENABLE ROW LEVEL SECURITY;
ALTER TABLE app.audit_log FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_rw ON app.projects USING (tenant_id = app.current_tenant()) WITH CHECK (tenant_id = app.current_tenant());
CREATE POLICY tenant_rw ON app.tasks    USING (tenant_id = app.current_tenant()) WITH CHECK (tenant_id = app.current_tenant());
CREATE POLICY tenant_read ON app.audit_log FOR SELECT USING (tenant_id = app.current_tenant());
CREATE POLICY audit_insert ON app.audit_log FOR INSERT WITH CHECK (true);

GRANT SELECT ON app.tenants TO tasks_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON app.projects, app.tasks TO tasks_app;
GRANT SELECT ON app.audit_log TO tasks_app;

-- search API: a security-invoker function, so RLS applies to the caller
CREATE FUNCTION app.search_tasks(q text, max_rows int DEFAULT 10)
RETURNS TABLE (id bigint, title text, status text, rank real, snippet text)
LANGUAGE sql STABLE AS $$
  SELECT t.id, t.title, t.status, ts_rank_cd(t.search, query) AS rank,
         ts_headline('english', t.body, query, 'MaxWords=12, MinWords=4') AS snippet
  FROM app.tasks t, websearch_to_tsquery('english', q) query
  WHERE t.search @@ query
  ORDER BY rank DESC, t.id
  LIMIT max_rows
$$;
RESET ROLE;
```

Design decisions:

- **Composite primary keys `(tenant_id, id)`**, and the task → project foreign key includes
  `tenant_id`. That makes it structurally impossible for a task to point at another tenant's project —
  the database rejects it even if the RLS policy were somehow bypassed (see step 3).
- `app.current_tenant()` wraps the `nullif(current_setting(...), '')` fix from lesson 7, so every policy
  uses the same, correct expression. It is `STABLE`, so the planner evaluates it once per query and
  can use it in index conditions.
- The audit trigger function is `SECURITY DEFINER` with a pinned `search_path`: it writes to
  `app.audit_log`, which the application role cannot write to directly (it has only `SELECT`). The
  `audit_insert` policy lets the trigger's inserts through; tenants can read only their own history.
- `to_jsonb(NEW) - 'search'` keeps the bulky `tsvector` out of the audit log.
- `app.search_tasks` is an ordinary (security-invoker) SQL function, so RLS applies to whoever calls it.

### Error 1: a trigger WHEN clause and generated columns

My first version reused lesson 4's `WHEN (OLD.* IS DISTINCT FROM NEW.*)` and failed:

```text
psql:01_setup.sql:62: ERROR:  BEFORE trigger's WHEN condition cannot reference NEW generated columns
LINE 2:   WHEN (OLD.* IS DISTINCT FROM NEW.*) EXECUTE FUNCTION app.t...
                                       ^
DETAIL:  A whole-row reference is used and the table contains generated columns.
```

In a BEFORE trigger, generated columns have not been computed yet, and `NEW.*` includes the generated
`search` column. Listing the real columns, as in the final version above, fixes it.

## Step 2 — seed data (as a superuser, which bypasses RLS)

```sql
-- as superuser (bypasses RLS)
SELECT setseed(0.3);
INSERT INTO app.tenants (name, plan) VALUES ('Acme', 'pro'), ('Globex', 'free'), ('Initech', 'pro');
INSERT INTO app.projects (tenant_id, name)
SELECT t, p FROM generate_series(1, 3) t, unnest(ARRAY['Website', 'Mobile app', 'Billing', 'Internal tools']) p;
INSERT INTO app.tasks (tenant_id, project_id, title, body, status, labels)
SELECT p.tenant_id, p.id,
       (ARRAY['Fix','Write','Review','Refactor','Test','Deploy'])[1 + (g % 6)] || ' ' ||
       (ARRAY['login page','invoice export','search ranking','password reset','dashboard chart','webhook retries','dark mode','CSV import'])[1 + (g % 8)],
       'Details for task ' || g || ': ' ||
       (ARRAY['customers report a timeout when exporting invoices','the search results ignore accents','retries should back off exponentially','charts render slowly on mobile','reset emails land in spam'])[1 + (g % 5)],
       (ARRAY['todo','doing','done','done'])[1 + (g % 4)],
       CASE g % 4 WHEN 0 THEN '{bug}'::text[] WHEN 1 THEN '{feature}' WHEN 2 THEN '{bug,urgent}' ELSE '{chore}' END
FROM app.projects p, generate_series(1, 2500) g;
ANALYZE;
SELECT (SELECT count(*) FROM app.tasks) AS tasks, (SELECT count(*) FROM app.audit_log) AS audit_rows;
```

```text
 tasks | audit_rows
-------+------------
 30000 |      30012
```

Three tenants × four projects × 2,500 tasks. The audit trigger fired for every seeded row (12 projects
+ 30,000 tasks) — fine for a demo, but in a real backfill you would load with the trigger disabled
and record that you did.

### Error 2: array of arrays

My first seed used `(ARRAY[ARRAY['bug'], ARRAY['feature'], ARRAY['bug','urgent'], ARRAY['chore']])[n]`
to pick a label set and got `column "labels" is of type text[] but expression is of type text`.
PostgreSQL arrays are multi-dimensional, not arrays of arrays. A 2-D `text[]` subscripted once has type
`text`, and at run time it returns NULL rather than a row (`(ARRAY[ARRAY['a','b'], ARRAY['c','d']])[1]`
is NULL; the slice `[1:1]` gives `{{a,b}}`). Sub-arrays must also have matching lengths — mixing
`ARRAY['bug']` with `ARRAY['bug','urgent']` fails with "multidimensional arrays must have array
expressions with matching dimensions". The `CASE` expression is the simple fix.

## Step 3 — test everything as the application

Connect as `tasks_app` and behave like the application, setting the tenant and the acting user at the
start of every transaction with `set_config(..., true)` (transaction-local):

```text
SELECT count(*) AS tasks_without_tenant FROM app.tasks;
 tasks_without_tenant
----------------------
                    0
```

No tenant, no data — the system fails closed. As Acme (tenant 1):

```sql
BEGIN;
SELECT set_config('app.tenant_id', '1', true), set_config('app.actor', 'maya@acme.example', true);
SELECT p.name, count(*) FILTER (WHERE t.status <> 'done') AS open, count(*) AS total
FROM app.projects p JOIN app.tasks t ON t.tenant_id = p.tenant_id AND t.project_id = p.id
GROUP BY p.name ORDER BY p.name;
```

```text
      name      | open | total
----------------+------+-------
 Billing        | 1250 |  2500
 Internal tools | 1250 |  2500
 Mobile app     | 1250 |  2500
 Website        | 1250 |  2500
```

Only Acme's four projects; the query has no `WHERE tenant_id` at all.

**Search**, through the function:

```text
SELECT id, title, status, round(rank::numeric, 3) AS rank, snippet FROM app.search_tasks('invoice timeout', 3);
 id  |        title         | status | rank  |                    snippet
-----+----------------------+--------+-------+-----------------------------------------------
 289 | Write invoice export | doing  | 0.197 | <b>timeout</b> when exporting <b>invoices</b>
 290 | Write invoice export | doing  | 0.197 | <b>timeout</b> when exporting <b>invoices</b>
 291 | Write invoice export | doing  | 0.197 | <b>timeout</b> when exporting <b>invoices</b>
```

(Identical scores because the seed data repeats; `t.id` breaks the ties deterministically.)

**Labels** with the GIN-indexed array:

```text
SELECT count(*) AS urgent_bugs FROM app.tasks WHERE labels @> '{bug,urgent}';
 urgent_bugs
-------------
        2500
```

**An update, its trigger and its audit entry:**

```text
UPDATE app.tasks SET status = 'doing' WHERE id = (SELECT min(id) FROM app.tasks WHERE status = 'todo')
RETURNING id, status, updated_at > created_at AS touched;
 id | status | touched
----+--------+---------
 37 | doing  | t

SELECT op, row_id, changed, actor FROM app.audit_log WHERE at > now() - interval '1 minute' AND op = 'UPDATE';
   op   | row_id |                                changed                                |       actor
--------+--------+-----------------------------------------------------------------------+-------------------
 UPDATE |     37 | {"status": "doing", "updated_at": "2026-10-11T12:42:06.653077+05:30"} | maya@acme.example
```

The audit records only the changed columns and the end user, not the shared database role.

**Attacks from inside tenant 1:**

```text
UPDATE app.tasks SET title = 'pwned' WHERE tenant_id = 2;
UPDATE 0

INSERT INTO app.tasks (tenant_id, project_id, title) VALUES (2, 5, 'planted');
ERROR:  new row violates row-level security policy for table "tasks"

-- (new transaction, still tenant 1) a task pointing at project 5, which belongs to tenant 2
INSERT INTO app.tasks (tenant_id, project_id, title) VALUES (1, 5, 'task in a project of tenant 2');
ERROR:  insert or update on table "tasks" violates foreign key constraint "tasks_tenant_id_project_id_fkey"
DETAIL:  Key is not present in table "projects".
```

The last attempt passes the RLS check (its `tenant_id` is 1) and is stopped by the composite foreign key
— defence in depth.

**As Globex (tenant 2):**

```text
 globex_tasks | pwned
--------------+-------
        10000 |     0

SELECT count(*) AS globex_sees_acme_audit FROM app.audit_log WHERE tenant_id = 1;
 globex_sees_acme_audit
------------------------
                      0

SELECT count(*) AS globex_audit_rows FROM app.audit_log;
 globex_audit_rows
-------------------
             10004
```

Globex's data is untouched, and it sees only its own audit history (its 10,000 tasks and 4 projects).

**The application cannot erase history:**

```text
DELETE FROM app.audit_log;
ERROR:  permission denied for table audit_log
```

**Partition pruning on the audit log** still works with the RLS condition added:

```text
EXPLAIN SELECT count(*) FROM app.audit_log WHERE at >= '2026-11-01';
 Aggregate
   ->  Seq Scan on audit_log_2026_11 audit_log
         Filter: ((at >= '2026-11-01 00:00:00+05:30'::timestamp with time zone)
                  AND (tenant_id = (NULLIF(current_setting('app.tenant_id'::text, true), ''::text))::bigint))
```

## Step 4 — a performance surprise: RLS and leakproof operators

Search as the application took about 2.5 ms — but the plan was not what I expected:

```text
EXPLAIN (ANALYZE) SELECT count(*) FROM app.tasks
WHERE search @@ websearch_to_tsquery('english', 'password reset spam');      -- as tasks_app, tenant 1
 Aggregate (actual time=2.264..2.264 rows=1.00 loops=1)
   ->  Bitmap Heap Scan on tasks (actual time=0.338..2.256 rows=252.00 loops=1)
         Recheck Cond: (tenant_id = (NULLIF(current_setting('app.tenant_id'::text, true), ''::text))::bigint)
         Filter: (search @@ '''password'' & ''reset'' & ''spam'''::tsquery)
         Rows Removed by Filter: 9748
         ->  Bitmap Index Scan on tasks_project_idx
               Index Cond: (tenant_id = (...))
```

The GIN index on `search` was **not used**. The planner fetched all 10,002 of the tenant's tasks and
tested each against the query. The same query as a superuser (no RLS) with an explicit
`tenant_id = 1`:

```text
   ->  Bitmap Heap Scan on tasks (actual time=3.380..3.570 rows=252.00 loops=1)
         Recheck Cond: ((tenant_id = 1) AND (search @@ '''password'' & ''reset'' & ''spam'''::tsquery))
         ->  BitmapAnd
               ->  Bitmap Index Scan on tasks_project_idx
               ->  Bitmap Index Scan on tasks_search_idx
```

The reason is in the catalog:

```text
 oprname |   oprcode   | proleakproof
---------+-------------+--------------
 @@      | ts_match_vq | f
```

Under RLS, user-supplied conditions whose functions are not marked `LEAKPROOF` must not run before the
policy's own condition — a leaky function could reveal data from rows the policy hides (lesson 7). So
`@@` cannot be used to drive an index scan ahead of the tenant filter. (Integer `=` *is* leakproof,
which is why the `tenant_id` index works.)

With 10,000 tasks per tenant this costs 2 ms. With 10 million it would be a problem. Options, in order
of preference:

1. Keep per-tenant row counts in mind when designing; for large tenants, search through a
   `SECURITY DEFINER` function that applies the tenant filter explicitly
   (`WHERE tenant_id = app.current_tenant() AND search @@ q`) — the function runs as the owner, so make
   sure it is the *only* way in and is written carefully.
2. Move search to a separate index store if it dominates.
3. A superuser can mark an operator's function `LEAKPROOF`; do that only after convincing yourself the
   function can never raise an error or otherwise reveal anything about its input.

The lesson generalises: **test performance as the application role, not as a superuser**, because the
plans differ.

## How It Actually Works

Every query from `tasks_app` is rewritten to include `tenant_id = app.current_tenant()` as a security
barrier qualification on each RLS table it touches. Because `app.current_tenant()` is `STABLE` and
the comparison operator is leakproof, the planner treats it like a parameterised index condition and
uses the `(tenant_id, …)` indexes; partition pruning on `at` happens independently of the policy.
Non-leakproof user predicates are placed above the barrier, which decides whether GIN indexes can help.

Writes pass three independent gates: the role's table privileges, the policy's `WITH CHECK`, and
ordinary constraints including the composite foreign key. The audit trigger then writes with its
owner's privileges, through a policy that allows inserts but not reads across tenants.

## Exercise

1. Build the project from these scripts. Add a `comments` table under the same tenancy, policy and audit
   pattern, with a full-text index on comment bodies.
2. Write a test script that, for each of the three tenants, asserts zero visibility of the other two
   tenants' rows in every table and that every cross-tenant write fails. Make it exit non-zero on any
   failure, so it can run in CI.
3. Implement the `SECURITY DEFINER` search function from step 4 and compare plans and timings for a
   tenant with 1 million tasks.
4. Add next month's audit partition automatically from a procedure, and drop partitions older than 12
   months.
