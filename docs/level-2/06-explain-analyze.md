---
description: "Read EXPLAIN and EXPLAIN ANALYZE output: costs, estimated vs actual rows, loops, buffers, join and sort methods, spills, and DML safety."
---

# 06 · Reading EXPLAIN ANALYZE

`EXPLAIN` shows the plan PostgreSQL *intends* to use. `EXPLAIN ANALYZE` runs the query and shows
what *actually happened* at every step. Being able to read that output fluently is the single most
valuable performance skill in PostgreSQL — every tuning decision in this course starts from it.

The examples use the 1-million-row `orders` table from lesson 4 and a 50,001-row `customers`
table, on PostgreSQL 18.6. Timings are from one run on a laptop; read them for proportions.

## The shape of a plan

A plan is a tree of **nodes**. Each node pulls rows from its children, does something to them, and
passes rows up. Read it from the **innermost, most indented** nodes outwards — that is where data
enters.

```text
EXPLAIN SELECT * FROM orders WHERE customer_id = 4242;
 Bitmap Heap Scan on orders  (cost=4.59..86.09 rows=21 width=54)
   Recheck Cond: (customer_id = 4242)
   ->  Bitmap Index Scan on orders_customer_incl_idx  (cost=0.00..4.58 rows=21 width=0)
         Index Cond: (customer_id = 4242)
```

The numbers in brackets are **estimates**:

- **cost=4.59..86.09** — startup cost (before the first row can be returned) and total cost, in
  arbitrary planner units (roughly "sequential page reads"). Only useful for comparing plans for the
  same query; the planner picks the lowest total.
- **rows=21** — estimated rows this node will output.
- **width=54** — estimated average row size in bytes.

## Adding ANALYZE

```text
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 4242;
 Bitmap Heap Scan on orders  (cost=4.59..86.09 rows=21 width=54) (actual time=0.334..0.926 rows=16.00 loops=1)
   Recheck Cond: (customer_id = 4242)
   Heap Blocks: exact=16
   Buffers: shared read=19
   ->  Bitmap Index Scan on orders_customer_incl_idx  (cost=0.00..4.58 rows=21 width=0) (actual time=0.187..0.188 rows=16.00 loops=1)
         Index Cond: (customer_id = 4242)
         Index Searches: 1
         Buffers: shared read=3
 Planning Time: 0.022 ms
 Execution Time: 0.933 ms
```

- **actual time=0.334..0.926** — milliseconds until the first row and until the last row, *per
  loop*.
- **rows=16.00** — actual rows output, *per loop* (shown with decimals since PostgreSQL 18, because
  it is an average across loops).
- **loops=1** — how many times the node ran.
- **Buffers** — pages touched. `shared hit` came from PostgreSQL's buffer cache; `shared read` had to
  be requested from the operating system (which may still have had it cached). Since PostgreSQL 18
  buffers are shown by default with `ANALYZE`; on older versions write `EXPLAIN (ANALYZE, BUFFERS)`.

Estimated 21 rows, actual 16: close enough. **Comparing estimated with actual rows is the first
thing to do in any slow plan** — when they differ by orders of magnitude, the planner chose based on
a wrong picture, and lesson 7 is about fixing that.

## Loops: multiply before you judge

```text
EXPLAIN (ANALYZE) SELECT c.name, o.total
FROM customers c JOIN orders o ON o.customer_id = c.id WHERE c.id < 4;
 Nested Loop  (cost=0.71..23.35 rows=60 width=20) (actual time=0.005..0.011 rows=46.00 loops=1)
   Buffers: shared hit=13
   ->  Index Scan using customers_pkey on customers c  (...) (actual time=0.001..0.002 rows=3.00 loops=1)
         Index Cond: (id < 4)
   ->  Index Only Scan using orders_customer_incl_idx on orders o  (...) (actual time=0.001..0.002 rows=15.33 loops=3)
         Index Cond: (customer_id = c.id)
         Heap Fetches: 0
         Index Searches: 3
```

The inner node ran **3 times** (once per customer) and returned **15.33 rows on average** — about 46
in total, which matches the join's output. A node showing `actual time=0.5 ms ... loops=20000` costs
10 seconds. Always multiply time and rows by loops.

## Join methods

| Node | How it works | Good when |
|------|--------------|-----------|
| **Nested Loop** | for each outer row, look up matches in the inner side (ideally via an index) | outer side is small; inner side has an index on the join key |
| **Hash Join** | build a hash table from the smaller input, then probe it with each row of the other | large unsorted inputs, equality joins |
| **Merge Join** | walk two inputs sorted on the join key in step | both inputs already sorted (e.g. from indexes), or very large inputs |

The previous plan was a nested loop: 3 customers, an index on the orders side. Aggregate every order
by country and the planner switches to a hash join:

```text
EXPLAIN (ANALYZE) SELECT c.country, count(*), sum(o.total)
FROM customers c JOIN orders o ON o.customer_id = c.id GROUP BY c.country;
 Finalize GroupAggregate  (...) (actual time=134.696..135.361 rows=5.00 loops=1)
   ->  Gather Merge  (...) (actual time=134.689..135.351 rows=15.00 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Sort  (...) (actual time=130.310..130.311 rows=5.00 loops=3)
               Sort Key: c.country
               Sort Method: quicksort  Memory: 25kB
               ->  Partial HashAggregate  (...) (actual time=130.286..130.287 rows=5.00 loops=3)
                     Group Key: c.country
                     Batches: 1  Memory Usage: 32kB
                     ->  Hash Join  (...) (actual time=7.131..93.044 rows=333333.33 loops=3)
                           Hash Cond: (o.customer_id = c.id)
                           ->  Parallel Seq Scan on orders o  (...) (actual time=0.115..12.879 rows=333333.33 loops=3)
                           ->  Hash  (...) (actual time=6.958..6.959 rows=50001.00 loops=3)
                                 Buckets: 65536  Batches: 1  Memory Usage: 2466kB
                                 ->  Seq Scan on customers c  (...) (actual time=0.024..3.782 rows=50001.00 loops=3)
 Execution Time: 135.397 ms
```

Reading from the inside out:

1. Each of three processes (the leader plus two workers, hence `loops=3`) scans all 50,001 customers
   and builds its own hash table in 2.4 MB — `Batches: 1` means it fit in memory.
2. Each process scans a third of `orders` (the *Parallel* Seq Scan) and probes the hash table.
3. Each process partially aggregates by country (`Partial HashAggregate`), producing 5 rows.
4. Gather Merge collects the 15 partial rows in sorted order; Finalize GroupAggregate combines them.

Most of the time is inside the hash join and its scan (93 ms of the 135 ms by the time the join
finished in each process). That is the profile of a query doing exactly what it was asked —
aggregate a million rows — and the only real improvements would be precomputation (a materialised
view or summary table) or doing less work.

## Sorts and memory: the disk spill

Each sort and hash node may use up to `work_mem` of memory before spilling to temporary files.
Forcing a large sort with a small `work_mem`:

```text
SET work_mem = '1MB';
EXPLAIN (ANALYZE) SELECT * FROM orders ORDER BY total DESC OFFSET 500000 LIMIT 1;
 Limit  (...) (actual time=412.160..426.271 rows=1.00 loops=1)
   Buffers: shared hit=372 read=10613, temp read=21387 written=25666
   ->  Gather Merge  (...)
         ->  Sort  (...) (actual time=322.441..338.326 rows=167069.67 loops=3)
               Sort Key: total DESC
               Sort Method: external merge  Disk: 23096kB
               Worker 0:  Sort Method: external merge  Disk: 22488kB
               Worker 1:  Sort Method: external merge  Disk: 22456kB
 Execution Time: 432.591 ms
```

`external merge  Disk: 23096kB` and `temp read/written` mean the sort spilled. With enough memory:

```text
SET work_mem = '200MB';
         ->  Sort  (...) (actual time=271.666..277.283 rows=167051.00 loops=3)
               Sort Method: quicksort  Memory: 59530kB
 Execution Time: 366.627 ms
```

In-memory quicksort, no temp I/O. (Only ~15% faster here because the temp files stayed in the OS
cache on a fast SSD; on a busy server with real disk I/O, spills hurt far more.) Look for the same
signal in hash nodes: `Batches: 8` instead of `Batches: 1` means a hash join or aggregate spilled.

Do not respond by raising `work_mem` globally: it is a *per-node, per-process* limit, so one complex
query in each of 200 connections can use many multiples of it. Raise it for the session or role that
runs the heavy report (`SET work_mem = '256MB'` or `ALTER ROLE reporting SET work_mem = ...`).
Level 4 · 01 covers memory sizing.

## Other lines worth knowing

- **Rows Removed by Filter** — rows read and then thrown away. Large numbers next to a small output
  suggest a missing or unusable index.
- **Rows Removed by Index Recheck / Heap Blocks: lossy** — a bitmap became too large for
  `work_mem` and degraded to whole pages, or the index is lossy by nature (BRIN, some GiST).
- **Heap Fetches** — on an index-only scan, rows that needed a heap visit (lesson 4).
- **Index Searches** (new in 18) — how many times an index was descended; skip scans and `IN` lists
  show more than 1.
- **Workers Planned vs Launched** — fewer launched than planned means the parallel worker pool
  (`max_parallel_workers`) was exhausted.
- **Planning Time** — high values (tens of milliseconds) on simple queries point at very many
  partitions, huge `IN` lists or missing catalog statistics.

## EXPLAIN ANALYZE executes the statement

```sql
BEGIN;
EXPLAIN (ANALYZE) UPDATE orders SET total = total WHERE customer_id = 4242;
ROLLBACK;
```

```text
 Update on orders  (...) (actual time=8.989..8.990 rows=0.00 loops=1)
   Buffers: shared hit=292 read=97 dirtied=77
   ->  Bitmap Heap Scan on orders  (...) (actual time=0.013..0.034 rows=16.00 loops=1)
```

The update really happened (`dirtied=77`) — the `ROLLBACK` undid it. **Always wrap `EXPLAIN ANALYZE`
of INSERT/UPDATE/DELETE in a transaction you roll back**, and remember side effects outside the
database (sequences advancing, triggers calling `NOTIFY`) still occur.

`ANALYZE` also discards the result rows without sending them to the client, so it hides the cost of
converting and transmitting them. PostgreSQL 17 added `SERIALIZE` to include that:

```text
EXPLAIN (ANALYZE, SERIALIZE, BUFFERS OFF, TIMING OFF) SELECT * FROM orders WHERE customer_id < 100;
 Bitmap Heap Scan on orders  (cost=23.98..5137.95 rows=2007 width=54) (actual rows=1979.00 loops=1)
 ...
 Serialization: output=192kB  format=text
 Execution Time: 6.304 ms
```

`TIMING OFF` skips per-node clock reads, which can noticeably inflate measurements on queries with
millions of node calls.

## A checklist for a slow plan

1. Where is the time? Find the node with the largest `actual time × loops` that its children do not
   explain.
2. Do estimated and actual rows agree at that node and below? If not, fix statistics first.
3. Is a lot of data read and discarded (`Rows Removed by Filter`, big `Buffers` for few rows)? An
   index, or a better one, may help.
4. Any spill (`external merge`, `Batches` > 1, `temp written`)?
5. Is a nested loop running many thousands of loops? A misestimate may have chosen it.
6. Is the plan fine, and the query simply asking for too much work?

For queries you cannot reproduce by hand, the `auto_explain` module logs plans of statements slower
than a threshold (Level 4 · 06), and tools such as explain.depesz.com or explain.dalibo.com
visualise long plans (they are third-party websites — do not paste plans containing sensitive
values).

## How It Actually Works

The executor is a tree of nodes implementing a common interface: initialise, *get next tuple*, end.
The top node is asked for a row, which asks its children, recursively — a demand-driven "Volcano"
model. That is why a `Limit` can stop a whole plan early, and why startup cost matters: a plan with
a sort must consume all input before producing its first row, while an index scan can produce the
first row immediately.

`EXPLAIN ANALYZE` wraps each node with instrumentation that counts rows and loops and, unless
`TIMING OFF`, reads the clock on every call. Buffer counters come from the shared buffer manager:
each request is counted as a hit if the page is already in `shared_buffers`, otherwise as a read
(which, from PostgreSQL's point of view, is a system call that the OS might satisfy from its page
cache). Times for parallel nodes are per process, averaged across loops — so parallel plans need a
little care: `loops=3` with a time of 130 ms means three processes each spent ~130 ms concurrently,
not 390 ms of wall time.

## Common mistakes

- Reading costs as milliseconds.
- Forgetting to multiply by `loops`.
- `EXPLAIN ANALYZE DELETE ...` outside a transaction on production.
- Testing plans on a tiny development database; plans depend on table sizes and statistics.
- Running a query twice and treating the second (cached) timing as the real one, or the first
  (cold) one — know which you are measuring and look at buffers, not only time.
- Raising `work_mem` globally to fix one report.

## Exercise

1. Run `EXPLAIN (ANALYZE)` on a three-table join from your own schema. Label every node with its
   join method and explain why the planner chose it.
2. Find a query that spills to disk at `work_mem = '1MB'`. Find the smallest `work_mem` at which it
   stops spilling.
3. Take a nested-loop plan with an inner index scan, drop the index, and compare the new plan's
   join method and buffers.
4. Use `EXPLAIN (ANALYZE, TIMING OFF)` and `EXPLAIN (ANALYZE)` on a query over a million rows and
   compare execution time. How much overhead does per-node timing add on your machine?
