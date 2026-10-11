---
description: "Isolate tenants with row-level security: USING and WITH CHECK policies, per-transaction settings, owner and view bypasses, pool leaks."
---

# 07 · Row-Level Security for Multi-Tenant Apps

In a multi-tenant SaaS application, every query must include `WHERE tenant_id = ?`. Every one, in
every endpoint, report and background job, forever. The day someone forgets, one customer sees
another's data. **Row-level security (RLS)** moves that filter into the database: once a policy is in
place, the database adds the condition to every query on the table, whether the application remembered
or not.

RLS is powerful and full of sharp edges. This lesson builds tenant isolation step by step and walks
through four real ways it leaked or broke while I built it, on PostgreSQL 18.6.

## Setup: an owner and an application role

As in Level 1 · 05, the table belongs to a non-login owner role and the application connects as a
separate role. That separation matters more than usual here, as you will see.

```sql
CREATE ROLE saas_owner NOLOGIN;
CREATE ROLE saas_app LOGIN PASSWORD 'app-dev-pw';
CREATE SCHEMA saas AUTHORIZATION saas_owner;
GRANT USAGE ON SCHEMA saas TO saas_app;

SET ROLE saas_owner;
CREATE TABLE saas.projects (
  id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id int NOT NULL,
  name      text NOT NULL
);
CREATE INDEX ON saas.projects (tenant_id);
INSERT INTO saas.projects (tenant_id, name) VALUES
  (1, 'Acme website'), (1, 'Acme app'), (2, 'Globex ERP'), (3, 'Initech TPS');
GRANT SELECT, INSERT, UPDATE, DELETE ON saas.projects TO saas_app;
```

## Enable RLS and add a policy

```sql
ALTER TABLE saas.projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON saas.projects
  USING      (tenant_id = current_setting('app.tenant_id', true)::int)
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::int);
```

- **`USING`** filters which existing rows are visible to `SELECT`, `UPDATE` and `DELETE`.
- **`WITH CHECK`** validates rows being written by `INSERT` and `UPDATE`.
- With RLS enabled and **no** policy that applies, a role sees nothing: the default is deny.

The tenant comes from a custom setting the application sets at the start of each transaction (the
same technique as the audit actor in lesson 4).

## It works

```text
SET ROLE saas_app;
SELECT * FROM saas.projects;
 id | tenant_id | name
----+-----------+------
(0 rows)
```

No tenant set — nothing visible. Now as tenant 1:

```sql
BEGIN;
SET LOCAL app.tenant_id = '1';
SELECT id, tenant_id, name FROM saas.projects ORDER BY id;
```

```text
 id | tenant_id |     name
----+-----------+--------------
  1 |         1 | Acme website
  2 |         1 | Acme app
```

Tenant 1 tries to reach tenant 2's data:

```text
UPDATE saas.projects SET name = 'hacked' WHERE tenant_id = 2;
UPDATE 0

INSERT INTO saas.projects (tenant_id, name) VALUES (2, 'planted in globex');
ERROR:  new row violates row-level security policy for table "projects"
```

The `UPDATE` cannot see tenant 2's rows, so it updates nothing; the `INSERT` is rejected by
`WITH CHECK`. The application's own `WHERE tenant_id = 2` was simply combined with the policy's
condition. Look at the plan:

```text
EXPLAIN SELECT * FROM saas.projects WHERE name LIKE 'Acme%';
 Bitmap Heap Scan on projects
   Recheck Cond: (tenant_id = (current_setting('app.tenant_id'::text, true))::integer)
   Filter: (name ~~ 'Acme%'::text)
   ->  Bitmap Index Scan on projects_tenant_id_idx
         Index Cond: (tenant_id = (current_setting('app.tenant_id'::text, true))::integer)
```

The policy is just another `WHERE` clause, and it uses the `tenant_id` index. **Index `tenant_id`** on
every RLS-protected table — usually as the leading column of composite indexes.

## Problem 1: the setting that turns into an empty string

After the transaction above committed, the owner's queries started failing. Isolated:

```text
SELECT current_setting('app.never_set', true) IS NULL AS unset_is_null;
 unset_is_null
---------------
 t

BEGIN; SET LOCAL app.tenant_id = '1'; COMMIT;
SELECT current_setting('app.tenant_id', true) AS after_local,
       current_setting('app.tenant_id', true) IS NULL AS is_null;
 after_local | is_null
-------------+---------
             | f
```

A setting that was **never** set returns NULL. One that was set with `SET LOCAL` and then reverted
returns an **empty string** for the rest of the session — and `''::int` is an error:

```text
ERROR:  invalid input syntax for type integer: ""
```

So the policy either errors or returns nothing depending on the connection's history — not a security
hole, but an outage waiting to happen. Fix the policy to treat empty as unset:

```sql
ALTER POLICY tenant_isolation ON saas.projects
  USING      (tenant_id = nullif(current_setting('app.tenant_id', true), '')::int)
  WITH CHECK (tenant_id = nullif(current_setting('app.tenant_id', true), '')::int);
```

## Problem 2: the table owner bypasses RLS

```text
SET ROLE saas_owner;
SELECT count(*) AS owner_sees FROM saas.projects;
 owner_sees
------------
          4
```

Policies do not apply to the table's **owner** — or to superusers, or roles with `BYPASSRLS`. If your
application connects as the owner (common in frameworks that run migrations and queries with the same
credentials), RLS silently does nothing. Two fixes, and you want both:

1. Never run application traffic as the owner (the role separation above).
2. `ALTER TABLE saas.projects FORCE ROW LEVEL SECURITY;` makes policies apply to the owner too:

```text
SELECT count(*) AS owner_sees_with_force FROM saas.projects;
 owner_sees_with_force
-----------------------
                     0
```

Superusers and `BYPASSRLS` roles still bypass everything — by design, so backups (`pg_dump`) and
maintenance work. `pg_dump` run by a non-bypassing role fails rather than produce a silently partial
dump unless you pass `--enable-row-security`.

## Problem 3: views run as their owner

A reporting view created by the owner:

```sql
SET ROLE saas_owner;
ALTER TABLE saas.projects NO FORCE ROW LEVEL SECURITY;   -- as many tables are configured
CREATE VIEW saas.project_names AS SELECT tenant_id, name FROM saas.projects;
GRANT SELECT ON saas.project_names TO saas_app;
```

The app, correctly scoped to tenant 1, queries the view:

```text
SELECT * FROM saas.project_names ORDER BY 1, 2;
 tenant_id |     name
-----------+--------------
         1 | Acme app
         1 | Acme website
         2 | Globex ERP
         3 | Initech TPS
```

**Every tenant's data.** A view accesses its tables with the privileges of the view's owner, and the
owner bypasses RLS. Since PostgreSQL 15, `security_invoker` makes the view check permissions and
policies as the querying user:

```sql
ALTER VIEW saas.project_names SET (security_invoker = true);
ALTER TABLE saas.projects FORCE ROW LEVEL SECURITY;
```

```text
SELECT * FROM saas.project_names ORDER BY 1, 2;
 tenant_id |     name
-----------+--------------
         1 | Acme app
         1 | Acme website
```

Audit every view, materialized view and `SECURITY DEFINER` function over RLS tables; each is a
potential bypass. (Materialized views have no `security_invoker` — they store the data as computed by
whoever refreshed them.)

## Problem 4: connection pools and session-level SET

The application must set the tenant **per transaction** (`SET LOCAL`, or
`SELECT set_config('app.tenant_id', $1, true)`). Here is what happens with a session-level `SET` on a
pooled connection:

```text
SET app.tenant_id = '1';
SELECT count(*) AS request_a_sees FROM saas.projects;
 request_a_sees
----------------
              2

-- request A finishes; the pool hands this connection to request B, which forgets to set a tenant
SELECT string_agg(name, ', ') AS request_b_sees FROM saas.projects;
     request_b_sees
------------------------
 Acme website, Acme app
```

Request B, which might be a different customer's request, sees tenant 1's data. With `SET LOCAL` the
setting dies at commit, and a request that forgets to set it sees nothing — failing closed. Make the
"set tenant" step part of the same code path that opens the transaction, so it cannot be skipped.

## More policy features

- Policies can be per command: `CREATE POLICY ... FOR SELECT`, `FOR INSERT`, etc., and per role:
  `TO support_staff`.
- Multiple **permissive** policies are OR-ed together; **restrictive** policies
  (`AS RESTRICTIVE`) are AND-ed with the result — useful for a global rule like "never show
  soft-deleted rows" on top of tenant isolation.
- Policies can use subqueries, e.g. `USING (project_id IN (SELECT project_id FROM memberships WHERE
  user_id = current_setting('app.user_id')::int))`. Keep them cheap and indexed; they run for every
  query.
- Functions called inside policies on user-supplied values can leak information through errors.
  PostgreSQL only pushes down quals it knows to be `LEAKPROOF` ahead of the policy check; non-leakproof
  operators in your `WHERE` clause are evaluated after it.

## How It Actually Works

Policies are stored in `pg_policy`. During query **rewriting**, for each table with RLS enabled (and a
user not exempt), PostgreSQL collects the applicable policies for the command type and role, ORs the
permissive ones, ANDs the restrictive ones, and adds the result as a **security barrier qualification**
on that table's scan. Security-barrier quals are evaluated before any user-supplied condition that is
not leakproof, so a user cannot write `WHERE leaky_function(secret_column)` to see rows the policy would
hide. `WITH CHECK` expressions become checks executed against each new row in the executor, like a
CHECK constraint that depends on session state.

Because policies are applied at rewrite time using the *current* user, views (rewritten as their
owner unless `security_invoker`) and `SECURITY DEFINER` functions (executed as their owner) change
whose policies apply — the mechanism behind problem 3.

## Common mistakes

- Application connecting as the table owner, without `FORCE ROW LEVEL SECURITY`.
- `current_setting(...)::int` without `nullif(..., '')`.
- Session-level `SET` of the tenant with a connection pooler.
- Views and materialized views over RLS tables created without `security_invoker`.
- Forgetting the `tenant_id` index, so every policy-filtered query scans the table.
- Treating RLS as the *only* control — keep tests that assert cross-tenant access fails.

## Exercise

1. Add a `tasks` table (with `tenant_id` and `project_id`) under the same policy pattern. Write a test
   script that, for each tenant, asserts it can see only its own projects and tasks and cannot insert
   or update into another tenant.
2. Add a restrictive policy that hides rows with `deleted_at IS NOT NULL` from the application role
   but not from a `support` role.
3. Measure the overhead of RLS: compare `EXPLAIN ANALYZE` of a tenant-scoped query with RLS and the
   same query with an explicit `WHERE tenant_id = 1` as the owner without `FORCE`.
4. Write a query against `pg_class`, `pg_policy` and `pg_views` that lists every table with RLS
   enabled but not forced, and every view over such tables lacking `security_invoker`.
