---
description: "PostgreSQL triggers in practice: BEFORE vs AFTER, an updated_at trigger, a JSONB audit log, transition tables, overhead, event triggers."
---

# 04 · Triggers & Audit Logging

A trigger runs a function automatically when rows change. That makes triggers ideal for rules that
must hold *no matter which code path changed the data*: keeping `updated_at` honest, writing an audit
trail, maintaining a summary. It also makes them easy to abuse — invisible side effects, hidden
performance costs and logic nobody remembers exists.

This lesson builds the two triggers almost every production schema has, measures what they cost,
and adds an event trigger that guards against an accidental `DROP TABLE`. Outputs are from
PostgreSQL 18.6.

## The four kinds of trigger

| | `FOR EACH ROW` | `FOR EACH STATEMENT` |
|---|---|---|
| **BEFORE** | can modify `NEW` or skip the row (`RETURN NULL`) | runs once before the statement |
| **AFTER** | sees the final row; return value ignored | runs once after; can read **transition tables** |

Plus `INSTEAD OF` triggers on views. Inside the function, `NEW` and `OLD` hold the row versions,
`TG_OP` is `'INSERT'`, `'UPDATE'`, `'DELETE'` or `'TRUNCATE'`, and `TG_TABLE_NAME` / `TG_TABLE_SCHEMA`
name the table — so one function can serve many tables.

## Trigger 1: updated_at that cannot lie

```sql
CREATE TABLE products (
  id int PRIMARY KEY, name text NOT NULL, price numeric(10,2) NOT NULL,
  updated_at timestamptz NOT NULL DEFAULT now());

CREATE FUNCTION touch_updated_at() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  NEW.updated_at := now();
  RETURN NEW;
END $$;

CREATE TRIGGER products_touch BEFORE UPDATE ON products
  FOR EACH ROW WHEN (OLD.* IS DISTINCT FROM NEW.*)
  EXECUTE FUNCTION touch_updated_at();
```

- It is a **BEFORE** trigger because it changes the row being written; a BEFORE row trigger returns
  the row to store (`NEW`), or `NULL` to silently skip it.
- The `WHEN` clause is evaluated before the function is called, so an `UPDATE` that changes nothing
  does not touch the timestamp — and does not pay for a function call.
- `now()` is the transaction start time. All rows updated in one transaction get the same value,
  which is usually what you want; `clock_timestamp()` gives the actual wall-clock time.

## Trigger 2: an audit log with diffs

```sql
CREATE SCHEMA audit;
CREATE TABLE audit.log (
  id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  table_name text NOT NULL,
  op         text NOT NULL,
  row_pk     text,
  changed    jsonb,
  old_row    jsonb,
  actor      text NOT NULL DEFAULT current_user,
  app_actor  text DEFAULT current_setting('app.actor', true),
  at         timestamptz NOT NULL DEFAULT clock_timestamp());

CREATE FUNCTION audit.capture() RETURNS trigger LANGUAGE plpgsql AS $$
DECLARE
  o jsonb := CASE WHEN TG_OP IN ('UPDATE','DELETE') THEN to_jsonb(OLD) END;
  n jsonb := CASE WHEN TG_OP IN ('UPDATE','INSERT') THEN to_jsonb(NEW) END;
  diff jsonb;
BEGIN
  IF TG_OP = 'UPDATE' THEN
    SELECT jsonb_object_agg(key, value) INTO diff
    FROM jsonb_each(n) WHERE o->key IS DISTINCT FROM value;
    IF diff IS NULL THEN RETURN NULL; END IF;     -- nothing actually changed
  ELSIF TG_OP = 'INSERT' THEN
    diff := n;
  END IF;
  INSERT INTO audit.log (table_name, op, row_pk, changed, old_row)
  VALUES (TG_TABLE_SCHEMA || '.' || TG_TABLE_NAME, TG_OP, coalesce(n, o)->>'id', diff, o);
  RETURN NULL;   -- AFTER trigger: return value ignored
END $$;

CREATE TRIGGER products_audit AFTER INSERT OR UPDATE OR DELETE ON products
  FOR EACH ROW EXECUTE FUNCTION audit.capture();
```

It is an **AFTER** trigger: the audit should record what was really written, after BEFORE triggers
and constraints have had their say. `to_jsonb(row)` turns any row into a document, so the same
function works for every table with an `id` column.

### Who made the change?

`current_user` is the database role — in most applications, the same pooled role for every request.
The real end user lives in the application. The usual technique is a custom setting the application
sets at the start of each transaction, which the trigger reads with `current_setting(name, true)`
(`true` = return NULL instead of erroring when unset):

```sql
BEGIN;
SET LOCAL app.actor = 'priya@shop.example';   -- or SELECT set_config('app.actor', $1, true)
UPDATE products SET price = 27.00 WHERE id = 1;
COMMIT;
```

Custom settings need a dotted name. My first attempt used `app.user`, which fails because `user` is a
reserved word:

```text
SET app.user = 'priya@shop.example';
ERROR:  syntax error at or near "user"
```

Use `SET LOCAL` (or `set_config(..., true)`) so the value disappears at commit — with a connection
pool, a session-level setting would leak into the next request that reuses the connection.

### What it records

```sql
INSERT INTO products (id, name, price) VALUES (2, 'Toaster', 45);
SET app.actor = 'priya@shop.example';
UPDATE products SET price = 27.00 WHERE id = 1;
UPDATE products SET price = price WHERE id = 2;      -- no real change
DELETE FROM products WHERE id = 2;
SELECT id, op, row_pk, changed, old_row->>'name' AS old_name, actor, app_actor FROM audit.log ORDER BY id;
```

```text
 id |   op   | row_pk |                                            changed                                            | old_name |  actor   |     app_actor
----+--------+--------+-----------------------------------------------------------------------------------------------+----------+----------+--------------------
 26 | INSERT | 2      | {"id": 2, "name": "Toaster", "price": 45.00, "updated_at": "2026-10-11T12:33:11.86997+05:30"} |          | postgres |
 27 | UPDATE | 1      | {"price": 27.00, "updated_at": "2026-10-11T12:33:11.871038+05:30"}                            | Kettle   | postgres | priya@shop.example
 28 | DELETE | 2      |                                                                                               | Toaster  | postgres | priya@shop.example
```

The update entry holds only the changed columns (including `updated_at`, set by the BEFORE trigger),
the no-op update produced nothing, and the delete keeps the whole old row. (The IDs start at 26
because I had truncated an earlier version of the log; `TRUNCATE` does not reset identity sequences
unless you add `RESTART IDENTITY`.)

My first version of this function lacked the `IF diff IS NULL` line, and every no-op `UPDATE` wrote an
audit row with an empty diff — exactly the noise that makes audit logs useless. The `WHEN` clause
trick from trigger 1 cannot be used here, because one trigger covers INSERT and DELETE too, where
`OLD` or `NEW` does not exist.

## Statement-level triggers with transition tables

Row triggers fire once per row. For "summarise what this statement did", a statement-level AFTER
trigger can see **all** changed rows at once as tables:

```sql
CREATE TABLE price_change_summary (at timestamptz DEFAULT now(), rows_changed int, avg_change numeric);

CREATE FUNCTION summarize_price_changes() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  INSERT INTO price_change_summary (rows_changed, avg_change)
  SELECT count(*), round(avg(n.price - o.price), 2)
  FROM new_rows n JOIN old_rows o USING (id)
  WHERE n.price <> o.price;
  RETURN NULL;
END $$;

CREATE TRIGGER products_price_summary AFTER UPDATE ON products
  REFERENCING OLD TABLE AS old_rows NEW TABLE AS new_rows
  FOR EACH STATEMENT EXECUTE FUNCTION summarize_price_changes();

UPDATE products SET price = price * 1.1 WHERE id >= 10;   -- ten products at 10.00
SELECT rows_changed, avg_change FROM price_change_summary;
```

```text
 rows_changed | avg_change
--------------+------------
           10 |       1.00
```

One function call and one set-based insert for the whole statement. Transition tables are the
efficient way to write audit logs for bulk operations, too: a 100,000-row `UPDATE` becomes one
`INSERT ... SELECT` into the log instead of 100,000 function calls.

## What triggers cost

Two identical tables, one with the row-level audit trigger, loaded with the same 100,000-row
`INSERT ... SELECT`:

```text
INSERT INTO plain_t   SELECT ... FROM generate_series(1, 100000) g;   Time: 218.811 ms
INSERT INTO audited_t SELECT ... FROM generate_series(1, 100000) g;   Time: 1042.504 ms
```

About five times slower: for each row, a PL/pgSQL call, a `to_jsonb`, and an insert into a second
table with its own index and WAL. For OLTP traffic of a few rows per transaction this is negligible;
for bulk loads and backfills it is not. Options: statement-level triggers with transition tables,
temporarily disabling an audit trigger during a controlled backfill
(`ALTER TABLE ... DISABLE TRIGGER audited_audit`, which needs owner rights and should itself be
logged), or logical decoding (Level 4 · 04) to capture changes outside the write path.

## Ordering, recursion and gotchas

- Multiple triggers of the same kind on a table fire in **alphabetical order of trigger name**.
  Name them deliberately (`10_validate`, `20_touch`) if order matters.
- A trigger that updates its own table fires itself again. Guard with `pg_trigger_depth()` or, better,
  avoid it.
- `TRUNCATE` does not fire row-level DELETE triggers; add a separate `ON TRUNCATE` statement trigger if
  your audit must see it.
- `COPY` fires row triggers like `INSERT`. Foreign-key cascades fire triggers on the child table.
- `session_replication_role = replica` disables ordinary triggers — used by logical replication and
  some bulk tools, and a way for a superuser to bypass your audit entirely. An audit trail written by
  triggers is tamper-*evident* only against ordinary users.

## Event triggers: reacting to DDL

Event triggers fire on DDL commands rather than row changes (`ddl_command_start`,
`ddl_command_end`, `sql_drop`, `table_rewrite`, and since PostgreSQL 17 `login`). A guard against
accidental drops in production:

```sql
CREATE FUNCTION guard_drops() RETURNS event_trigger LANGUAGE plpgsql AS $$
DECLARE obj record;
BEGIN
  FOR obj IN SELECT * FROM pg_event_trigger_dropped_objects() WHERE object_type = 'table' LOOP
    IF current_setting('app.allow_drop', true) IS DISTINCT FROM 'on' THEN
      RAISE EXCEPTION 'dropping table % is blocked; SET app.allow_drop = on in a migration',
                      obj.object_identity;
    END IF;
  END LOOP;
END $$;
CREATE EVENT TRIGGER no_accidental_drops ON sql_drop EXECUTE FUNCTION guard_drops();
```

```text
DROP TABLE plain_t;
ERROR:  dropping table public.plain_t is blocked; SET app.allow_drop = on in a migration
CONTEXT:  PL/pgSQL function guard_drops() line 6 at RAISE

SET app.allow_drop = on;
DROP TABLE plain_t;
DROP TABLE
```

The `sql_drop` event fires after the objects are dropped but before the transaction commits, so
raising an error rolls the drop back. Event triggers require superuser to create, and a broken one
can block all DDL — keep them small and test them.

## How It Actually Works

Triggers are rows in `pg_trigger` pointing at a function. During execution, each modified row passes
through the trigger manager: BEFORE row triggers run inline as the executor forms the new tuple (so
they can change it); AFTER row triggers are *queued* in the AFTER-trigger event list along with the
tuple identities, and fired at the end of the statement (or at commit, for deferrable constraint
triggers — which is how foreign keys are implemented). That queue lives in memory and spills to disk
for very large statements, which is part of why row triggers on bulk operations are expensive.

For transition tables, the executor additionally stores every affected old and new tuple in
tuplestores for the statement, which the trigger function then reads like ordinary tables named by
the `REFERENCING` clause.

Event triggers hook into the utility-command processing path; `pg_event_trigger_ddl_commands()` and
`pg_event_trigger_dropped_objects()` expose what the current command did.

## Common mistakes

- Business logic hidden in triggers that application developers do not know exist.
- Row-level audit triggers on tables that receive large bulk loads, with no plan for backfills.
- Recording only `current_user` when every request uses the same pooled role.
- Session-level `SET` of the actor with a connection pool.
- Audit rows for updates that changed nothing.
- Triggers that modify their own table and recurse.

## Exercise

1. Attach `audit.capture()` to two more tables and write a query that reconstructs the full history of
   one row from `audit.log`.
2. Rewrite the audit as statement-level triggers using transition tables (you need one trigger per
   operation). Measure the 100,000-row insert again.
3. Add a BEFORE INSERT OR UPDATE trigger that normalises `email` to lower case and trims spaces, then
   compare it with the generated-column approach from Level 1 · 10. When would you choose each?
4. Write an event trigger on `ddl_command_end` that records every DDL command (type, object, role,
   time) into an `audit.ddl_log` table.
