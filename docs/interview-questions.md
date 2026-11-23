---
title: "PostgreSQL Interview Questions and Answers (179 Questions)"
description: "179 PostgreSQL interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the PostgreSQL Mastery Path."
---

# PostgreSQL Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free PostgreSQL Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · PostgreSQL Foundations

### [01 · How PostgreSQL Is Built: Clusters, Processes &amp; Setup](level-1/01-architecture-and-setup.md)

**Q1.** Which PostgreSQL process accepts client connections and forks a backend per connection?

**Answer:** The postmaster

Process-per-connection model.

**Q2.** What is a PostgreSQL cluster?

**Answer:** One server instance managing a data directory with many databases

initdb creates it.

**Q3.** Which file controls host-based client authentication?

**Answer:** pg_hba.conf

Order matters: first match wins.

**Q4.** Where does a new database's template come from?

**Answer:** template1 (copied by CREATE DATABASE)

template0 is the pristine one.

### [02 · psql Like a Pro](level-1/02-psql-like-a-pro.md)

**Q5.** Which psql command lists tables in the current schema search path?

**Answer:** \dt

\l lists databases, \dn schemas.

**Q6.** What does \x on do in psql?

**Answer:** Switches to expanded (vertical) output

Useful for wide rows.

**Q7.** How do you run a SQL file in psql?

**Answer:** \i file.sql (or psql -f file.sql)

Add -v ON_ERROR_STOP=1 for scripts.

**Q8.** What does \timing show?

**Answer:** How long each statement took on the client side

Toggle on/off.

### [03 · PostgreSQL Data Types That Matter](level-1/03-data-types.md)

**Q9.** What is the result?

```sql
select 7/2 as a, 7/2.0 as b, 7 % 3 as c, 2^10 as d
```

**Answer:** 3,3.5000000000000000,1,1024

Integer division truncates; 2^10 is double precision. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q10.** What is the result?

```sql
select 0.1::float8 + 0.2::float8 = 0.3::float8 as f, 0.1::numeric + 0.2::numeric = 0.3::numeric as n
```

**Answer:** f,t

Use numeric for money. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q11.** What is the result?

```sql
select 'abc'::varchar(2) is null as x, length('héllo'), octet_length('héllo')
```

**Answer:** f,5,6

Characters vs bytes (UTF-8). Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q12.** What error appears?

```sql
select 'abcd'::varchar(3)::int + 1
```

**Answer:** invalid input syntax for type integer: "abc"

An explicit cast to varchar(3) truncates, then the int cast fails. Verified by running it on PostgreSQL 18 (the error message is shown).

### [04 · Schemas, search_path &amp; Organizing Objects](level-1/04-schemas-search-path.md)

**Q13.** What is the result?

```sql
create schema a; create schema b;
create table a.t(x int); create table b.t(x int);
insert into a.t values (1); insert into b.t values (2);
set search_path = b, a;

select x from t
```

**Answer:** 2

The first schema in search_path wins. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q14.** Why is it safer to set search_path explicitly on SECURITY DEFINER functions?

**Answer:** So a caller can't shadow objects with their own schema

CVE-class issue.

**Q15.** What does CREATE SCHEMA give you?

**Answer:** A namespace for tables and objects, useful for organisation and permissions

One database, many schemas.

**Q16.** Where is the public schema's default CREATE privilege since PG 15?

**Answer:** Revoked from PUBLIC; only the database owner has it

Safer default.

### [05 · Roles, Privileges &amp; Client Authentication](level-1/05-roles-privileges.md)

**Q17.** What is a role in PostgreSQL?

**Answer:** A unified user/group concept; roles can own objects and have privileges

LOGIN makes a role a user.

**Q18.** What is the result?

```sql
create role r1; create table t(x int); grant select on t to r1;

select has_table_privilege('r1','t','select') as s, has_table_privilege('r1','t','insert') as i
```

**Answer:** t,f

Grants are explicit. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q19.** What does ALTER DEFAULT PRIVILEGES do?

**Answer:** Sets privileges applied automatically to objects created in the future

Otherwise new tables aren't readable.

**Q20.** Which privilege does a role need to see rows in a table in another schema?

**Answer:** USAGE on the schema plus SELECT on the table

Two levels.

### [06 · Constraints Beyond the Basics](level-1/06-constraints.md)

**Q21.** What error appears?

```sql
create table t(a int not null, b int check (b > 0), unique(a));
insert into t values (1, 5);

insert into t values (1, 7)
```

**Answer:** duplicate key value violates unique constraint "t_a_key"

Constraint names are auto-generated. Verified by running it on PostgreSQL 18 (the error message is shown).

**Q22.** What error appears?

```sql
create table t(a int not null, b int check (b > 0));

insert into t values (1, 0)
```

**Answer:** new row for relation "t" violates check constraint "t_b_check"

CHECK rejects false (not null). Verified by running it on PostgreSQL 18 (the error message is shown).

**Q23.** What is the result?

```sql
create table t(a int, b int check (b > 0));
insert into t values (1, null);

select count(*) from t
```

**Answer:** 1

CHECK passes when the expression is NULL. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q24.** What error appears?

```sql
create table p(id int primary key); create table c(p int references p(id));
insert into p values (1);

delete from p where id = 1; insert into c values (2)
```

**Answer:** insert or update on table "c" violates foreign key constraint "c_p_fkey"

The delete is fine; the insert references a missing parent. Verified by running it on PostgreSQL 18 (the error message is shown).

### [07 · Transactions &amp; Isolation Levels in Practice](level-1/07-transactions-isolation.md)

**Q25.** What is the result?

```sql
create table a(id int primary key, bal int); insert into a values (1,100);

begin; update a set bal = 50 where id = 1; rollback; select bal from a
```

**Answer:** 100

Rollback discards. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q26.** What is the result?

```sql
create table a(id int primary key, bal int); insert into a values (1,100);

begin; update a set bal = 1; savepoint s; update a set bal = 2; rollback to s; commit; select bal from a
```

**Answer:** 1

Savepoints undo part of a transaction. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q27.** What error does the final SELECT report?

```sql
create table a(id int primary key);
insert into a values (1);

begin;
insert into a values (1);
select 1
```

**Answer:** current transaction is aborted, commands ignored until end of transaction block

After an error the transaction is aborted until ROLLBACK. Verified by running it on PostgreSQL 18 (the error message is shown).

**Q28.** Which level does PostgreSQL use by default?

**Answer:** Read Committed

Snapshot per statement.

### [08 · PostgreSQL-Flavoured SQL: RETURNING, Upserts, MERGE &amp; More](level-1/08-postgres-sql-features.md)

**Q29.** What is the result?

```sql
create table inv(sku text primary key, qty int);
insert into inv values ('A',5);

insert into inv values ('A',3),('B',7) on conflict (sku) do update set qty = inv.qty + excluded.qty returning sku, qty
```

**Answer:** A,8; B,7

Upsert adds to the existing row. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q30.** What is the result?

```sql
create table t(id serial primary key, n text);

insert into t(n) values ('x'),('y') returning id, n
```

**Answer:** 1,x; 2,y

RETURNING gives generated values. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q31.** What is the result?

```sql
create table e(user_id int, ts int);
insert into e values (1,10),(1,20),(2,5),(2,7);

select distinct on (user_id) user_id, ts from e order by user_id, ts desc
```

**Answer:** 1,20; 2,7

DISTINCT ON keeps the first row per group by ORDER BY. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q32.** What is the result?

```sql
create table o(s text);
insert into o values ('ok'),('ok'),('bad');

select count(*) filter (where s = 'ok') as good, count(*) as total from o
```

**Answer:** 2,3

FILTER on aggregates. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

### [09 · Backups with pg_dump &amp; pg_restore](level-1/09-pg-dump-restore.md)

**Q33.** What does pg_dump -Fc produce?

**Answer:** A compressed custom-format archive restored with pg_restore

Allows selective, parallel restore.

**Q34.** What does pg_dump NOT back up?

**Answer:** Cluster-wide objects like roles (use pg_dumpall --globals-only)

Separate step.

**Q35.** What is a logical backup's trade-off versus physical?

**Answer:** Portable across versions but slower to restore and no point-in-time recovery

Use base backups + WAL for PITR.

**Q36.** What proves a backup works?

**Answer:** A regular test restore

Untested backups are hopes.

### [10 · Project — Event Ticketing Database](level-1/10-project-event-ticketing.md)

**Q37.** What does the event ticketing project combine?

**Answer:** Schema design, constraints, transactions and upserts for a booking system

Level 1 project.

**Q38.** Two buyers race for the last seat. What prevents double-booking?

```sql
create table seat(id int primary key, sold boolean not null default false);
insert into seat values (1,false);

update seat set sold = true where id = 1 and not sold returning id
```

**Answer:** 1

A conditional update is atomic: only one session gets a row back. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q39.** What constraint stops two bookings for the same seat and show?

**Answer:** A unique constraint on (show_id, seat_id)

Database enforces invariants.

**Q40.** How should ticket totals be stored?

**Answer:** numeric (or integer cents), never float

Exact arithmetic.

## Level 2 · Internals: MVCC, Storage & Indexes

### [01 · MVCC: How Rows Become Visible](level-2/01-mvcc.md)

**Q41.** What is the result?

```sql
create table t(id int primary key, v int);
insert into t values (1, 1);

select xmin::text::bigint > 0 as has_xmin, count(*) over () as n from t
```

**Answer:** t,1

Every row version has xmin/xmax. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q42.** What does MVCC give readers?

**Answer:** A consistent snapshot without blocking writers (and vice versa)

Multiple row versions.

**Q43.** What does an UPDATE do physically?

**Answer:** Creates a new row version and marks the old one dead

Dead tuples need vacuum.

**Q44.** When can a dead row version be removed?

**Answer:** When no running transaction can still see it

Long transactions hold back cleanup.

### [02 · VACUUM, Autovacuum &amp; Bloat](level-2/02-vacuum-autovacuum.md)

**Q45.** What does VACUUM do?

**Answer:** Reclaims space from dead tuples for reuse and updates visibility info

Plain VACUUM doesn't return space to the OS.

**Q46.** What triggers autovacuum on a table?

**Answer:** Dead tuples exceeding threshold + scale factor × rows

Tune per table.

**Q47.** What does a long-running idle-in-transaction session cause?

**Answer:** It holds back vacuum cleanup and causes bloat

Set timeouts.

**Q48.** What prevents transaction ID wraparound?

**Answer:** (Auto)vacuum freezing old rows

Monitor age(datfrozenxid).

### [03 · Pages, TOAST, Fillfactor &amp; HOT Updates](level-2/03-storage-toast-hot.md)

**Q49.** What is TOAST?

**Answer:** Out-of-line storage for large column values, compressed and chunked

Automatic for values &gt; ~2 kB.

**Q50.** What is a HOT update?

**Answer:** An update that doesn't change indexed columns and stays on the same page, avoiding index writes

Helped by fillfactor.

**Q51.** Why lower fillfactor on update-heavy tables?

**Answer:** To leave room on pages for HOT updates

Trade space for speed.

**Q52.** What is the result?

```sql
create table big(id int, s text); insert into big select g, repeat('x', 5000) from generate_series(1,100) g; analyze big;

select count(*) from big where length(s) > 4000
```

**Answer:** 100

Large values are TOASTed transparently. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

### [04 · B-tree Indexes in Depth](level-2/04-btree-indexes.md)

**Q53.** What is the result?

```sql
create table t(a int primary key, b int);
insert into t select g, g%10 from generate_series(1,1000) g;
create index on t(b); analyze t;
set enable_seqscan = off;

explain (costs off) select * from t where a = 5
```

**Answer:** Index Scan using t_pkey on t; Index Cond: (a = 5)

Primary key lookups use the B-tree. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q54.** Which operators can a B-tree index serve?

**Answer:** =, &lt;, &lt;=, &gt;, &gt;=, BETWEEN and prefix LIKE with the right collation

Ordered data.

**Q55.** What is a composite index's column order rule?

**Answer:** Equality columns first, then range; leftmost prefix matters

Design for queries.

**Q56.** What does a partial index save?

**Answer:** Space and write cost by indexing only rows matching a predicate

WHERE status = 'open'.

### [05 · GIN, GiST, BRIN &amp; Hash Indexes](level-2/05-other-index-types.md)

**Q57.** When use a GIN index?

**Answer:** For multi-valued data: jsonb, arrays, full-text

Inverted index.

**Q58.** When use a BRIN index?

**Answer:** On huge, naturally ordered tables (e.g. timestamps) for tiny summaries

Block range.

**Q59.** When use GiST?

**Answer:** Geometric, range types and exclusion constraints, nearest neighbour

Flexible framework.

**Q60.** When is a hash index useful?

**Answer:** Pure equality lookups (crash-safe since PG 10)

Rarely better than B-tree.

### [06 · Reading EXPLAIN ANALYZE](level-2/06-explain-analyze.md)

**Q61.** Which plan does the planner choose for a selective predicate on an unindexed column?

```sql
create table t(id int primary key, g int);
insert into t select i, i%5 from generate_series(1,1000) i; analyze t;

explain (costs off) select * from t where g = 3
```

**Answer:** Seq Scan on t; Filter: (g = 3)

No index on g, so a sequential scan with a filter. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q62.** What does EXPLAIN ANALYZE do that EXPLAIN doesn't?

**Answer:** Actually executes the statement and reports real timings and row counts

Wrap writes in a transaction you roll back.

**Q63.** What is a big gap between estimated and actual rows a sign of?

**Answer:** Stale or insufficient statistics (or correlated columns)

Run ANALYZE / extended statistics.

**Q64.** What does BUFFERS add to EXPLAIN ANALYZE?

**Answer:** Shared buffer hits and reads, showing I/O behaviour

Cache vs disk.

### [07 · The Planner &amp; Statistics](level-2/07-planner-statistics.md)

**Q65.** What does ANALYZE collect?

**Answer:** Column statistics (histograms, most common values) for the planner

Autovacuum runs it too.

**Q66.** How improve estimates for correlated columns?

**Answer:** CREATE STATISTICS (extended statistics) on those columns

Functional dependencies.

**Q67.** What does default_statistics_target control?

**Answer:** The size of the sample/histogram per column

Raise for skewed columns.

**Q68.** What does random_page_cost reflect?

**Answer:** The relative cost of non-sequential I/O; lower it on SSDs

Planner tuning.

### [08 · Locks, Deadlocks &amp; SKIP LOCKED](level-2/08-locking.md)

**Q69.** What is the result?

```sql
create table t(id int primary key, v int); insert into t values (1,0);

begin; select v from t where id = 1 for update; select count(*) from pg_locks where relation = 't'::regclass and mode = 'RowShareLock'
```

**Answer:** 0; 1

The first SELECT returns v (0); FOR UPDATE also takes a RowShareLock on the table, which the second query counts (1). Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q70.** What causes a deadlock?

**Answer:** Transactions waiting on each other's locks in a cycle; PostgreSQL aborts one

Lock in consistent order.

**Q71.** What does lock_timeout do?

**Answer:** Aborts a statement that waits too long for a lock

Use for migrations.

**Q72.** What does SKIP LOCKED enable?

**Answer:** Queue-style workers that grab unlocked rows without waiting

FOR UPDATE SKIP LOCKED.

### [09 · WAL, Checkpoints &amp; Durability](level-2/09-wal-checkpoints.md)

**Q73.** What is WAL?

**Answer:** The write-ahead log: changes are logged before data files are modified

Crash safety and replication.

**Q74.** What does a checkpoint do?

**Answer:** Flushes dirty buffers so recovery needn't replay older WAL

Spikes in I/O.

**Q75.** What does synchronous_commit = off risk?

**Answer:** Losing the last few commits on a crash (no corruption)

Trade durability for speed.

**Q76.** Which setting controls checkpoint frequency by WAL size?

**Answer:** max_wal_size

Tune with checkpoint_timeout.

### [10 · Project — Diagnose a Slow Database](level-2/10-project-slow-database.md)

**Q77.** What does the slow database project teach?

**Answer:** Finding, explaining and fixing slow queries with indexes, statistics and vacuum

Level 2 project.

**Q78.** Where do you find the slowest queries?

**Answer:** pg_stat_statements ordered by total time

Extension.

**Q79.** What do you check after adding an index?

**Answer:** That the plan uses it and the write overhead is acceptable

Measure.

**Q80.** What is the first fix for an N+1 pattern?

**Answer:** Fetch in one query with a join or IN list

Reduce round trips.

## Level 3 · Advanced Features

### [01 · JSONB in Depth](level-3/01-jsonb.md)

**Q81.** What is the result?

```sql
create table d(doc jsonb); insert into d values ('{"a":1,"tags":["x","y"],"n":{"k":true}}');

select doc->>'a', doc->'tags'->>1, doc#>>'{n,k}', doc ? 'a', doc @> '{"a":1}' from d
```

**Answer:** 1,y,true,t,t

Operators -&gt;, -&gt;&gt;, #&gt;&gt;, ? and @&gt;. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q82.** What is the result?

```sql
select jsonb_build_object('a',1,'b',null) as j, jsonb_typeof('[1]'::jsonb), '{"a":1}'::jsonb || '{"b":2}'::jsonb
```

**Answer:** {"a": 1, "b": null},array,{"a": 1, "b": 2}

jsonb normalises formatting. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q83.** Which index type speeds up @&gt; and ? on jsonb?

**Answer:** GIN

jsonb_path_ops for @&gt; only.

**Q84.** When should data be columns rather than jsonb?

**Answer:** When it's relational, constrained and queried regularly

Use jsonb for flexible attributes.

### [02 · Full-Text Search](level-3/02-full-text-search.md)

**Q85.** What is the result?

```sql
select to_tsvector('english','The quick brown foxes jumped') @@ to_tsquery('english','fox & jump') as m, plainto_tsquery('english','running dogs')::text
```

**Answer:** t,'run' &amp; 'dog'

Stemming and stop words. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q86.** What speeds full-text search?

**Answer:** A GIN index on the tsvector (generated column)

Store tsvector.

**Q87.** What does ts_rank do?

**Answer:** Scores how well a document matches the query

ORDER BY rank DESC.

**Q88.** When use pg_trgm instead?

**Answer:** For fuzzy/substring/typo-tolerant matching (LIKE '%x%', similarity)

Trigram GIN/GiST.

### [03 · PL/pgSQL Functions &amp; Procedures](level-3/03-plpgsql.md)

**Q89.** What is the result?

```sql
create function add2(a int, b int) returns int language plpgsql as $$ begin return a + b; end $$;

select add2(2,3), (select count(*) from generate_series(1,add2(1,1)))
```

**Answer:** 5,2

A simple PL/pgSQL function. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q90.** What is the result?

```sql
create function f(n int) returns int language plpgsql as $$ declare t int := 1; begin for i in 1..n loop t := t * i; end loop; return t; end $$;

select f(5), f(0)
```

**Answer:** 120,1

Loops and variables. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q91.** What error does RAISE EXCEPTION produce?

```sql
create function chk(n int) returns void language plpgsql as $$ begin if n < 0 then raise exception 'negative: %', n; end if; end $$;

select chk(-3)
```

**Answer:** negative: -3

Custom errors. Verified by running it on PostgreSQL 18 (the error message is shown).

**Q92.** When is a function better than application code?

**Answer:** For set-based logic near the data that must be atomic and reused

Balance with testability.

### [04 · Triggers &amp; Audit Logging](level-3/04-triggers-audit.md)

**Q93.** What is the result?

```sql
create table acc(id int primary key, bal int); create table audit(acc int, old int, new int);
create function lg() returns trigger language plpgsql as $$ begin insert into audit values (old.id, old.bal, new.bal); return new; end $$;
create trigger t after update on acc for each row execute function lg();
insert into acc values (1,100);
update acc set bal = 70 where id = 1;

select * from audit
```

**Answer:** 1,100,70

AFTER UPDATE audit trigger. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q94.** What does a BEFORE trigger returning NULL do?

**Answer:** Skips the operation for that row

Return NEW to proceed.

**Q95.** What is a trade-off of triggers?

**Answer:** Hidden logic that can surprise and slow bulk operations

Document them.

**Q96.** Which is better for audit tables?

**Answer:** Append-only audit rows written by triggers inside the same transaction

Atomic with the change.

### [05 · Advanced Querying: LATERAL, Recursive CTEs &amp; Grouping Sets](level-3/05-advanced-querying.md)

**Q97.** What is the result?

```sql
create table s(g text, v int); insert into s values ('a',1),('a',3),('b',2),('b',5),('b',4);

select g, v, rank() over (partition by g order by v desc) as r, sum(v) over (partition by g) as tot from s order by g, v desc
```

**Answer:** a,3,1,4; a,1,2,4; b,5,1,11; b,4,2,11; b,2,3,11

Window functions keep every row. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q98.** What is the result?

```sql
with recursive n(i) as (select 1 union all select i+1 from n where i < 5) select sum(i), count(*) from n
```

**Answer:** 15,5

Recursive CTE. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q99.** What is the result?

```sql
create table t(a int); insert into t values (1),(2),(3),(4);

select a, lag(a) over (order by a) as prev, lead(a,2) over (order by a) as nxt from t
```

**Answer:** 1,,3; 2,1,4; 3,2,; 4,3,

lag/lead return NULL at the edges. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q100.** What does LATERAL allow?

```sql
create table u(id int); insert into u values (1),(2);

select u.id, x.v from u cross join lateral (select u.id * 10 as v) x order by 1
```

**Answer:** 1,10; 2,20

The subquery can reference earlier FROM items. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

### [06 · Declarative Partitioning](level-3/06-partitioning.md)

**Q101.** What does declarative partitioning do?

**Answer:** Splits a table into child tables by range, list or hash behind one name

Partition pruning.

**Q102.** What is the result?

```sql
create table m(d date, v int) partition by range (d);
create table m1 partition of m for values from ('2026-01-01') to ('2026-02-01');
create table m2 partition of m for values from ('2026-02-01') to ('2026-03-01');
insert into m values ('2026-01-15',1),('2026-02-10',2);

select tableoid::regclass::text, v from m order by v
```

**Answer:** m1,1; m2,2

Rows land in matching partitions. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q103.** What error appears?

```sql
create table m(d date, v int) partition by range (d);
create table m1 partition of m for values from ('2026-01-01') to ('2026-02-01');

insert into m values ('2027-01-01',1)
```

**Answer:** no partition of relation "m" found for row

Add a default partition if needed. Verified by running it on PostgreSQL 18 (the error message is shown).

**Q104.** Main operational benefit of partitioning?

**Answer:** Fast retention (drop old partitions) and smaller indexes

Not automatic speedup.

### [07 · Row-Level Security for Multi-Tenant Apps](level-3/07-row-level-security.md)

**Q105.** What is the result?

```sql
create table doc(id int, owner text); insert into doc values (1,'ann'),(2,'bob'),(3,'ann');
create role viewer; grant select on doc to viewer;
alter table doc enable row level security;
create policy p on doc for select to viewer using (owner = current_setting('app.uid'));
set role viewer; set app.uid = 'ann';

select id from doc order by id
```

**Answer:** 1; 3

RLS filters rows per policy. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q106.** Do table owners bypass RLS by default?

**Answer:** Yes, unless FORCE ROW LEVEL SECURITY is set

Superusers and BYPASSRLS always bypass.

**Q107.** What's the key setup for multi-tenant RLS?

**Answer:** Policies comparing tenant_id to a session setting, set per request

Defence in depth.

**Q108.** What's a risk with RLS and connection pooling?

**Answer:** Session settings leaking between clients; use SET LOCAL inside transactions

Transaction pooling.

### [08 · Extensions Worth Knowing](level-3/08-extensions.md)

**Q109.** How do you enable an extension?

**Answer:** CREATE EXTENSION name; (the package must be installed on the server)

Per database.

**Q110.** What is the result?

```sql
create extension pg_trgm;

select similarity('postgres','postgras') > 0.4 as sim, 'postgres' % 'postgras' as m
```

**Answer:** t,t

Trigram similarity. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q111.** What is the result?

```sql
create extension pgcrypto;

select length(gen_random_uuid()::text), encode(digest('abc','sha256'),'hex') like 'ba7816bf%'
```

**Answer:** 36,t

pgcrypto and built-in UUIDs. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q112.** Which extension gives per-statement stats?

**Answer:** pg_stat_statements

Needs shared_preload_libraries.

### [09 · LISTEN/NOTIFY &amp; Job Queues in Postgres](level-3/09-listen-notify-queues.md)

**Q113.** What does LISTEN/NOTIFY give you?

**Answer:** Lightweight in-database pub/sub notifications delivered after commit

Payload ≤ 8000 bytes.

**Q114.** Why combine NOTIFY with a table?

**Answer:** Notifications aren't durable; the table is the source of truth

Wake-up signal.

**Q115.** What makes a safe job queue query?

**Answer:** SELECT … FOR UPDATE SKIP LOCKED with a status update in one transaction

Concurrent workers.

**Q116.** What limits LISTEN with pooling?

**Answer:** Transaction-pooled connections can't hold a LISTEN session

Use a dedicated connection.

### [10 · Project — Multi-Tenant SaaS Backend](level-3/10-project-multi-tenant-saas.md)

**Q117.** What does the multi-tenant SaaS project combine?

**Answer:** Tenant isolation (RLS), partitioning, jsonb settings, audit triggers and indexes

Level 3 project.

**Q118.** Where must tenant_id appear?

**Answer:** In every table, key and index used for tenant-scoped queries

Composite keys.

**Q119.** How enforce tenant isolation reliably?

**Answer:** RLS policies in the database plus application checks

Defence in depth.

**Q120.** What about noisy tenants?

**Answer:** Per-tenant limits, statement timeouts and partitioning for large ones

Fairness.

## Level 4 · Operations & Production

### [01 · Configuration &amp; Memory Tuning](level-4/01-configuration-tuning.md)

**Q121.** Which setting is typically ~25% of RAM?

**Answer:** shared_buffers

Starting point.

**Q122.** What is work_mem per?

**Answer:** Each sort/hash operation in each query, so total can multiply

Be conservative.

**Q123.** What does effective_cache_size do?

**Answer:** Tells the planner how much OS+PG cache to assume; allocates nothing

Planner hint.

**Q124.** How change settings safely?

**Answer:** One at a time with measurement, via ALTER SYSTEM/config files

Check pg_settings.context.

### [02 · Connections &amp; Pooling](level-4/02-connection-pooling.md)

**Q125.** Why use a connection pooler?

**Answer:** Each connection is a process; poolers multiplex many clients over few connections

PgBouncer.

**Q126.** What does PgBouncer transaction pooling break?

**Answer:** Session state such as SET, prepared statements (older versions), LISTEN, advisory locks

Know the limits.

**Q127.** How size max_connections?

**Answer:** Modestly; use a pool rather than thousands of backends

Memory per backend.

**Q128.** What does pool_mode=session keep?

**Answer:** Full session semantics for the client's whole connection

Fewer savings.

### [03 · Streaming Replication](level-4/03-streaming-replication.md)

**Q129.** What does streaming replication ship?

**Answer:** WAL records to standbys that replay them (physical copy)

Byte-identical standby.

**Q130.** What does a hot standby allow?

**Answer:** Read-only queries on the replica

Offload reads.

**Q131.** What is synchronous replication's trade-off?

**Answer:** Commits wait for a standby, adding latency but reducing data-loss risk

synchronous_standby_names.

**Q132.** What is replication lag?

**Answer:** How far the standby's replay is behind the primary

Monitor it.

### [04 · Logical Replication &amp; Change Data Capture](level-4/04-logical-replication.md)

**Q133.** What does logical replication replicate?

**Answer:** Row changes of selected tables via publications/subscriptions

Cross-version possible.

**Q134.** Does logical replication copy schema?

**Answer:** No; create the schema on the subscriber first

DDL isn't replicated.

**Q135.** Good use case?

**Answer:** Zero-downtime major upgrades and selective data sharing

Flexible.

**Q136.** What needs a replica identity?

**Answer:** UPDATE/DELETE on tables need a primary key or REPLICA IDENTITY

Default is PK.

### [05 · Point-in-Time Recovery](level-4/05-point-in-time-recovery.md)

**Q137.** What does PITR need?

**Answer:** A base backup plus continuous WAL archiving

Restore to any time.

**Q138.** What does recovery_target_time do?

**Answer:** Stops WAL replay at a chosen timestamp

Recover before the mistake.

**Q139.** What's a common PITR mistake?

**Answer:** Never testing restores or letting WAL archiving silently fail

Monitor archive_command.

**Q140.** What tools help?

**Answer:** pgBackRest or Barman for managed base backups and WAL

Retention and verification.

### [06 · Monitoring a Running Database](level-4/06-monitoring.md)

**Q141.** Which view shows current activity?

**Answer:** pg_stat_activity

Find long queries.

**Q142.** What to alert on?

**Answer:** Replication lag, connection saturation, bloat, long transactions and disk

Actionable.

**Q143.** What does pg_stat_user_tables help find?

**Answer:** Tables with dead tuples and heavy sequential scans

Maintenance signals.

**Q144.** What do log_min_duration_statement logs give?

**Answer:** Slow statements with durations for analysis

Feed pgBadger.

### [07 · Zero-Downtime Schema Migrations](level-4/07-zero-downtime-migrations.md)

**Q145.** Which operation locks a table longest when adding a column with volatile default?

**Answer:** Older versions rewrote the table; modern ones add constant defaults without rewrite

Check version behaviour.

**Q146.** How add an index without blocking writes?

**Answer:** CREATE INDEX CONCURRENTLY

Can't run in a transaction.

**Q147.** What's the safe NOT NULL pattern?

**Answer:** Add a CHECK … NOT VALID, validate it, then set NOT NULL

Short locks.

**Q148.** Why set lock_timeout in migrations?

**Answer:** To fail fast instead of queuing behind long locks and blocking everyone

Retry.

### [08 · Security Hardening](level-4/08-security-hardening.md)

**Q149.** What's the principle for roles?

**Answer:** Least privilege with separate roles for owners, apps and read-only

Limit blast radius.

**Q150.** Which authentication method is preferred?

**Answer:** scram-sha-256 (with TLS)

Modern.

**Q151.** What do you do with the public schema?

**Answer:** Revoke CREATE from PUBLIC and restrict as needed

Hardening.

**Q152.** What encrypts data in transit?

**Answer:** TLS (ssl = on, sslmode=verify-full on clients)

Verify certificates.

### [09 · Upgrades &amp; High Availability](level-4/09-upgrades-high-availability.md)

**Q153.** How do minor upgrades work?

**Answer:** Replace binaries and restart; data files stay compatible

Do them promptly.

**Q154.** What does pg_upgrade --link do?

**Answer:** Upgrades a cluster in place using hard links for speed

Rollback caveats.

**Q155.** What does high availability need besides replicas?

**Answer:** Automated failover (Patroni etc.), fencing and tested runbooks

Avoid split-brain.

**Q156.** What gets tested regularly?

**Answer:** Failover and restore drills

Practice.

### [10 · Capstone — Production-Ready PostgreSQL](level-4/10-capstone-production-ready.md)

**Q157.** What does the production-ready capstone combine?

**Answer:** Hardening, tuning, backups/PITR, replication, monitoring and migrations

Level 4 project.

**Q158.** How prove readiness?

**Answer:** Restore drill, failover drill, load test and alerting review

Evidence.

**Q159.** What documentation is essential?

**Answer:** Runbooks for failover, restore, upgrades and incident response

On-call.

**Q160.** What sustains operations?

**Answer:** Patching, capacity reviews and ownership

Process.

## Mixed questions across all levels

**Q161.** What does this return?

```sql
create table t(a int, b int);
insert into t values (1,null),(2,5);

select count(*), count(b), sum(b), avg(b) from t
```

**Answer:** 2,1,5,5.0000000000000000

Aggregates ignore NULLs; count(*) counts rows. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q162.** What does this return?

```sql
create table t(a int);
insert into t values (1),(2),(3);

select a from t where a not in (select a from t where a > 5) order by a
```

**Answer:** 1; 2; 3

An empty subquery makes NOT IN true (NULLs would make it unknown). Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q163.** What does this return?

```sql
create table t(a int); insert into t values (1),(null);

select count(*) from t where a not in (select 1 union all select null)
```

**Answer:** 0

NOT IN with a NULL in the list never returns true. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q164.** What error appears?

```sql
create table t(a int);
insert into t values (1);

select a, count(*) from t
```

**Answer:** column "t.a" must appear in the GROUP BY clause or be used in an aggregate function

Non-aggregated columns need GROUP BY. Verified by running it on PostgreSQL 18 (the error message is shown).

**Q165.** What does this return?

```sql
create table t(id int primary key, v int);
insert into t values (1,10);

insert into t values (1,99) on conflict (id) do update set v = excluded.v returning id, v
```

**Answer:** 1,99

The upsert replaced the value. Verified by running it on PostgreSQL 18 (rows shown comma-separated, separated by semicolons).

**Q166.** Which anomaly can Read Committed show inside one transaction?

**Answer:** A non-repeatable read: the same query can return different results because each statement takes a new snapshot

Use Repeatable Read or Serializable for consistent multi-statement reads.

**Q167.** Why is a conditional UPDATE … WHERE not sold a safe way to claim a seat?

**Answer:** The row lock makes it atomic: only one concurrent transaction changes the row and sees it returned

Let the database enforce invariants.

**Q168.** What does MVCC mean for readers and writers?

**Answer:** Readers don't block writers and writers don't block readers (they see different row versions)

Snapshots.

**Q169.** Why does a table with many UPDATEs get bloated without vacuum?

**Answer:** Each update leaves a dead row version until vacuum reclaims it

Tune autovacuum.

**Q170.** What does EXPLAIN (ANALYZE, BUFFERS) tell you?

**Answer:** The real plan with timings, row counts and buffer hits/reads

Compare estimates to actuals.

**Q171.** A composite index on (a, b) is best for which query?

**Answer:** WHERE a = ? AND b &gt; ? (and WHERE a = ?)

Leftmost prefix.

**Q172.** Which index type fits jsonb containment (@&gt;) queries?

**Answer:** GIN

Inverted index.

**Q173.** How add an index to a busy table safely?

**Answer:** CREATE INDEX CONCURRENTLY

Avoids blocking writes.

**Q174.** What does the planner rely on to estimate row counts?

**Answer:** Statistics gathered by ANALYZE

Keep them fresh.

**Q175.** A worker queue with many consumers should claim jobs with?

**Answer:** SELECT … FOR UPDATE SKIP LOCKED

No double processing, no waiting.

**Q176.** What protects a multi-tenant table so one tenant can't read another's rows?

**Answer:** Row-level security policies on tenant_id (plus application checks)

Defence in depth.

**Q177.** What do you need for point-in-time recovery?

**Answer:** A base backup plus continuous WAL archiving

Practice restores.

**Q178.** What does a streaming standby replay?

**Answer:** WAL from the primary (physical, byte-identical copy)

Great for HA and read scaling.

**Q179.** Which setting avoids connection explosions?

**Answer:** A pooler (PgBouncer) with a modest max_connections

Process per connection.

## Practical coding questions

### Atomic seat booking and availability report

Finish the schema (a foreign key to the seat, a positive-price CHECK and a UNIQUE constraint so a seat can only be sold once), then claim seats with INSERT … ON CONFLICT DO NOTHING RETURNING so the second buyer of seat 2 gets no row, and produce the availability and revenue reports.

Starter code:

```
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

Expected output:

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

??? success "Show a solution"

    ```
    CREATE TABLE seat (
      show_id int NOT NULL,
      seat_no int NOT NULL,
      price_cents int NOT NULL CHECK (price_cents > 0),
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
    INSERT INTO booking (show_id, seat_no, customer) VALUES (1, 2, 'ann') ON CONFLICT DO NOTHING RETURNING id, seat_no, customer;
    INSERT INTO booking (show_id, seat_no, customer) VALUES (1, 2, 'bob') ON CONFLICT DO NOTHING RETURNING id, seat_no, customer;
    INSERT INTO booking (show_id, seat_no, customer) VALUES (1, 4, 'bob') ON CONFLICT DO NOTHING RETURNING id, seat_no, customer;

    -- Availability report: free seats and revenue so far.
    SELECT s.seat_no, COALESCE(b.customer, '(free)') AS status
    FROM seat s LEFT JOIN booking b USING (show_id, seat_no)
    WHERE s.show_id = 1
    ORDER BY s.seat_no;

    SELECT count(b.id) AS sold,
           count(*) - count(b.id) AS free,
           sum(s.price_cents) FILTER (WHERE b.id IS NOT NULL) AS revenue_cents
    FROM seat s LEFT JOIN booking b USING (show_id, seat_no)
    WHERE s.show_id = 1;
    ```

    The unique constraint makes the claim atomic, so a lost race simply returns no row. LEFT JOIN with FILTER computes sold, free and revenue in one pass. Verified by running the script on PostgreSQL 18 with psql.

### Window functions, recursive CTE and JSON aggregation

Write four queries: rank reps per region with a percentage of the regional total, a running total per region, an indented org chart with a recursive CTE, and a per-region jsonb object of rep totals.

Starter code:

```
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

Expected output:

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

??? success "Show a solution"

    ```
    CREATE TABLE sale (id serial PRIMARY KEY, region text, rep text, amount int, sold_on date);
    INSERT INTO sale (region, rep, amount, sold_on) VALUES
     ('north','ann',500,'2026-01-05'), ('north','bob',300,'2026-01-20'), ('north','ann',700,'2026-02-03'),
     ('south','cy',400,'2026-01-11'), ('south','cy',900,'2026-02-14'), ('south','dee',200,'2026-02-20');
    CREATE TABLE org (id int PRIMARY KEY, name text, boss int);
    INSERT INTO org VALUES (1,'zoe',NULL),(2,'ann',1),(3,'bob',1),(4,'cy',2),(5,'dee',2);

    -- 1. Rank reps within each region by total sales, with each rep's share of the region total.
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
    SELECT repeat('  ', depth) || name AS org_chart FROM tree ORDER BY depth, name;

    -- 4. Aggregate as JSON.
    SELECT region, jsonb_object_agg(rep, total ORDER BY rep) AS by_rep
    FROM (SELECT region, rep, sum(amount) AS total FROM sale GROUP BY region, rep) x
    GROUP BY region ORDER BY region;
    ```

    Window functions compute per-group rankings and running totals without collapsing rows, a recursive CTE walks the hierarchy, and jsonb_object_agg builds JSON in the database. Verified by running the script on PostgreSQL 18 with psql.

---

More free practice: the [PostgreSQL Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
