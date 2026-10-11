---
description: "Server-side logic in PostgreSQL: SQL vs PL/pgSQL functions, volatility, SQLSTATE errors, procedures that commit, and safe dynamic SQL."
---

# 03 · PL/pgSQL Functions & Procedures

PostgreSQL can run your code inside the database: in SQL itself, in PL/pgSQL (its procedural
language), and through extensions in Python, JavaScript, Rust and others. Server-side code is the
right tool when logic must be atomic with the data it touches, when it is used by several
applications, or when moving rows to the client and back would cost more than the work itself.
It is the wrong tool for business logic that changes weekly and needs unit tests, logging and
deploy pipelines your team already has for application code.

This lesson covers writing functions and procedures well. Outputs are from PostgreSQL 18.6.

## SQL functions first

If a function is a single query or expression, write it in plain SQL:

```sql
CREATE FUNCTION vat(amount numeric, rate numeric DEFAULT 0.18)
RETURNS numeric LANGUAGE sql IMMUTABLE PARALLEL SAFE
RETURN round(amount * rate, 2);

SELECT vat(100), vat(100, 0.05);
```

```text
  vat  | vat
-------+------
 18.00 | 5.00
```

The `RETURN expression` form (and `BEGIN ATOMIC ... END` for multi-statement bodies), available since
PostgreSQL 14, is parsed at creation time: dependencies on tables and functions are tracked, so you
cannot drop something the function uses by accident. Simple SQL functions can also be **inlined** into
the calling query, letting the planner optimise through them — PL/pgSQL functions are always a black
box.

A set-returning SQL function is a parameterised view:

```sql
CREATE FUNCTION account_statement(p_id int)
RETURNS TABLE (at timestamptz, direction text, amount numeric) LANGUAGE sql STABLE AS $$
  SELECT at, CASE WHEN from_id = p_id THEN 'out' ELSE 'in' END, amount
  FROM transfers WHERE p_id IN (from_id, to_id) ORDER BY at
$$;
SELECT direction, amount FROM account_statement(2);
```

## Volatility is a promise to the planner

Every function is `VOLATILE` (default), `STABLE` or `IMMUTABLE`:

| Category | Promise | Examples |
|----------|---------|----------|
| `IMMUTABLE` | same arguments → same result, forever; no database access | `lower()`, `vat()` above |
| `STABLE` | same result within one statement; may read the database | `now()`, lookups |
| `VOLATILE` | anything goes; may change between calls or have side effects | `random()`, anything that writes |

It changes plans. Two functions returning the same constant, one left at the default:

```text
EXPLAIN (ANALYZE) SELECT * FROM prices WHERE id = target_id_volatile();
 Seq Scan on prices (actual time=7.846..41.713 rows=1.00 loops=1)
   Filter: (id = target_id_volatile())
   Rows Removed by Filter: 199999

EXPLAIN (ANALYZE) SELECT * FROM prices WHERE id = target_id_stable();
 Index Scan using prices_pkey on prices (actual time=0.009..0.010 rows=1.00 loops=1)
   Index Cond: (id = target_id_stable())
```

A volatile function might return a different value for every row, so the planner must call it per
row and cannot use it as an index key. Mark functions as strictly as is *true* — and never lie:
an `IMMUTABLE` function that reads a table can return stale results from expression indexes and
cached plans. Only immutable functions can be used in index expressions and generated columns.

## PL/pgSQL: a transfer function with real error handling

```sql
CREATE FUNCTION transfer(p_from int, p_to int, p_amount numeric)
RETURNS bigint LANGUAGE plpgsql AS $$
DECLARE
  v_id bigint;
BEGIN
  IF p_amount <= 0 THEN
    RAISE EXCEPTION 'amount must be positive, got %', p_amount USING ERRCODE = '22023';
  END IF;
  -- lock both rows in a fixed order to avoid deadlocks
  PERFORM 1 FROM accounts WHERE id IN (p_from, p_to) ORDER BY id FOR UPDATE;
  UPDATE accounts SET balance = balance - p_amount WHERE id = p_from;
  IF NOT FOUND THEN
    RAISE EXCEPTION 'account % not found', p_from USING ERRCODE = 'P0002';
  END IF;
  UPDATE accounts SET balance = balance + p_amount WHERE id = p_to;
  IF NOT FOUND THEN
    RAISE EXCEPTION 'account % not found', p_to USING ERRCODE = 'P0002';
  END IF;
  INSERT INTO transfers (from_id, to_id, amount) VALUES (p_from, p_to, p_amount) RETURNING id INTO v_id;
  RETURN v_id;
EXCEPTION
  WHEN check_violation THEN
    RAISE EXCEPTION 'insufficient funds in account %', p_from
      USING ERRCODE = 'P0001', HINT = 'balance would go negative';
END;
$$;
```

Accounts start with balances 100 and 50, and `balance` has `CHECK (balance >= 0)`:

```text
SELECT transfer(1, 2, 30);
 transfer
----------
        1

SELECT transfer(1, 2, 500);
ERROR:  insufficient funds in account 1
HINT:  balance would go negative
CONTEXT:  PL/pgSQL function transfer(integer,integer,numeric) line 22 at RAISE

SELECT transfer(1, 99, 5);
ERROR:  account 99 not found
CONTEXT:  PL/pgSQL function transfer(integer,integer,numeric) line 16 at RAISE

SELECT transfer(1, 2, -5);
ERROR:  amount must be positive, got -5

SELECT id, owner, balance FROM accounts ORDER BY id;
 id | owner | balance
----+-------+---------
  1 | ana   |   70.00
  2 | ben   |   80.00
```

Points to notice:

- **A function runs inside the caller's transaction.** In the "account 99" case the debit from
  account 1 had already happened when the exception was raised; the error rolled it back, which is
  why ana still has 70.
- `FOUND` is set by the last SQL statement — the simplest way to detect "no row matched".
- Raise errors with a **SQLSTATE** (`USING ERRCODE`), so clients can branch on codes rather than
  message text. With verbose error output the code is visible:

    ```text
    \set VERBOSITY verbose
    ERROR:  P0001: insufficient funds in account 1
    HINT:  balance would go negative
    CONTEXT:  PL/pgSQL function transfer(integer,integer,numeric) line 22 at RAISE
    LOCATION:  exec_stmt_raise, pl_exec.c:3923
    ```

- An `EXCEPTION` block is implemented as a **subtransaction** (a savepoint) entered on every call.
  That costs something and consumes a subtransaction ID; in a function called millions of times,
  catch errors only where you need to translate or recover from them.
- `RAISE NOTICE` / `RAISE DEBUG` are for diagnostics, reaching the client and/or server log depending
  on `client_min_messages` and `log_min_messages`.

## Set-based beats row-by-row

The most common performance mistake in PL/pgSQL is writing a loop where one statement would do:

```sql
DO $$
DECLARE r record;
BEGIN
  FOR r IN SELECT id, amount FROM prices LOOP
    UPDATE prices SET with_tax = r.amount + vat(r.amount) WHERE id = r.id;
  END LOOP;
END $$;
-- Time: 1380.173 ms

UPDATE prices SET with_tax = amount + vat(amount);
-- Time: 897.406 ms
```

For 200,000 rows, inside the server, the loop was about 50% slower — 200,000 separate index lookups
and executor start-ups versus one pass. The gap is modest here only because PL/pgSQL statements do not
cross a network. The same loop written in application code pays a round trip per row; Level 1 · 08
measured 100,000 single-row inserts at 4.7 seconds even inside one transaction. Think in sets:
`UPDATE ... FROM`, `INSERT ... SELECT`, `MERGE`, CTEs with `RETURNING`.

## Procedures can commit

Functions cannot commit or roll back; **procedures** (`CREATE PROCEDURE`, invoked with `CALL`) can,
when called outside an explicit transaction block. That makes them the tool for batch maintenance
that must not hold locks or one giant transaction for its whole run:

```sql
CREATE PROCEDURE archive_old_transfers(batch_size int DEFAULT 1000)
LANGUAGE plpgsql AS $$
DECLARE moved int;
BEGIN
  LOOP
    WITH batch AS (
      DELETE FROM big_log
      WHERE id IN (SELECT id FROM big_log
                   WHERE created_at < now() - interval '30 days' LIMIT batch_size)
      RETURNING *)
    INSERT INTO big_log_archive SELECT * FROM batch;
    GET DIAGNOSTICS moved = ROW_COUNT;
    RAISE NOTICE 'moved % rows', moved;
    EXIT WHEN moved < batch_size;
    COMMIT;   -- release locks and let VACUUM reclaim as we go
  END LOOP;
END $$;

CALL archive_old_transfers(500);
```

```text
NOTICE:  moved 500 rows
NOTICE:  moved 500 rows
NOTICE:  moved 241 rows
CALL
```

Each batch is its own transaction: if the procedure is interrupted, completed batches stay done, and a
long-running archive does not hold back VACUUM (Level 2 · 02). Procedure transaction control only
works when the `CALL` is the top-level statement; inside an explicit transaction block:

```text
BEGIN;
CALL archive_old_transfers(500);
NOTICE:  moved 500 rows
ERROR:  invalid transaction termination
CONTEXT:  PL/pgSQL function archive_old_transfers(integer) line 12 at COMMIT
```

## Dynamic SQL without injection

`EXECUTE` runs a SQL string built at run time — needed when the table or column name is a parameter.
Building that string by concatenation is SQL injection waiting to happen:

```sql
CREATE FUNCTION count_rows_unsafe(tbl text) RETURNS bigint LANGUAGE plpgsql AS $$
DECLARE n bigint;
BEGIN
  EXECUTE 'SELECT count(*) FROM ' || tbl INTO n;
  RETURN n;
END $$;

SELECT count_rows_unsafe($$(SELECT 1 FROM pg_authid WHERE rolsuper AND rolname LIKE 'p%') AS x$$);
```

```text
 leaked
--------
      1
```

The "table name" was a subquery against the password catalog, and the count leaked whether a
superuser name starts with "p". Ask enough yes/no questions and you can extract anything the
function's role can read. Defences:

```sql
CREATE FUNCTION count_rows(tbl regclass) RETURNS bigint LANGUAGE plpgsql AS $$
DECLARE n bigint;
BEGIN
  EXECUTE format('SELECT count(*) FROM %s', tbl) INTO n;
  RETURN n;
END $$;

SELECT count_rows('public.accounts');                       -- 2
SELECT count_rows($$(SELECT 1 FROM pg_authid) AS x$$);
-- ERROR:  invalid name syntax
```

- Accept identifiers as `regclass` (or `regproc`, `regtype`): the input is validated as an existing
  object before your code runs, and its output form is safely quoted.
- Otherwise build strings with `format()`: `%I` for identifiers, `%L` for literals.
- Pass **values** with `EXECUTE ... USING`, never by concatenation:
  `EXECUTE format('SELECT count(*) FROM %I WHERE owner = $1', tbl) INTO n USING p_owner;`

## SECURITY DEFINER functions

By default a function runs with the privileges of the *caller* (`SECURITY INVOKER`). `SECURITY
DEFINER` runs it with the privileges of its *owner* — a controlled way to let a role do one specific
privileged thing. Because unqualified names are resolved at run time using the caller's
`search_path` (Level 1 · 04), such functions must pin it:

```sql
CREATE FUNCTION api.reset_my_password(...) ...
LANGUAGE plpgsql SECURITY DEFINER
SET search_path = pg_catalog, pg_temp
AS $$ ... $$;
REVOKE EXECUTE ON FUNCTION api.reset_my_password(...) FROM PUBLIC;
```

Functions are executable by `PUBLIC` by default, hence the `REVOKE`. Level 4 · 08 returns to this.

## How It Actually Works

A PL/pgSQL function's source is stored as text in `pg_proc.prosrc`. On the first call in a session,
the PL/pgSQL interpreter parses it into a tree of statements and caches it. Each embedded SQL
statement is prepared through the SPI (Server Programming Interface) on first execution and its plan
cached for the session — with the same custom-versus-generic plan logic as prepared statements
(Level 2 · 07), using the function's variables as parameters. Simple expressions like `x + 1` are
evaluated by a fast path that bypasses full query execution.

Because plans are cached per session, a function that worked yesterday can use a plan that is wrong
for today's data distribution until the session reconnects or something invalidates the plan (DDL on
a referenced table does). And because names in the SQL are resolved when each statement is first
prepared in a session, the caller's `search_path` at that moment decides which objects are used —
the root of the `SECURITY DEFINER` advice.

Procedures differ from functions in being invoked by `CALL` at the top level, where the executor can
end the current transaction and start a new one mid-execution; the procedure's local variables
survive across those commits, but open cursors other than the driving `FOR` loop's do not.

## Common mistakes

- Row-by-row loops where a single set-based statement would do.
- Leaving functions `VOLATILE` that could be `STABLE`, or marking functions `IMMUTABLE` that read
  tables.
- `EXCEPTION WHEN OTHERS THEN NULL` — swallowing every error, including ones you needed to see.
- Concatenating identifiers or values into `EXECUTE` strings.
- `SECURITY DEFINER` without a fixed `search_path` and without revoking `PUBLIC` execute.
- Expecting `COMMIT` inside a procedure called within an outer transaction to work.

## Exercise

1. Write `transfer` as a procedure instead of a function. What can the procedure do that the function
   could not, and what can it no longer do (hint: can you `SELECT` it)?
2. Write a set-returning SQL function `top_customers(n int, since timestamptz)` and compare
   `EXPLAIN` of a query calling it with the same query written inline. Is the function inlined?
3. Write a PL/pgSQL function that takes a table name and a column name and returns the number of
   NULLs in that column. Make it injection-proof, and prove it with a hostile input.
4. Add an exception handler to a function that is called once per row in a 1-million-row
   `UPDATE`, and measure the cost of the subtransactions compared with the same function without it.
