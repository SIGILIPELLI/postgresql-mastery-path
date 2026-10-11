---
description: "B-tree indexes in PostgreSQL: index, bitmap and index-only scans, column order, skip scan, expression, partial and covering indexes."
---

# 04 · B-tree Indexes in Depth

Almost every index you create in PostgreSQL is a B-tree, and most "my query is slow" problems end
with creating — or more often, correcting — one. Getting them right needs more than "index the
columns in the WHERE clause". Column order, expressions, partial predicates, sort direction and the
visibility map all change whether an index helps.

This lesson uses a 1-million-row `orders` table in a database called `perf`:

```sql
CREATE TABLE orders (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id int NOT NULL,
  status      text NOT NULL,
  email       text NOT NULL,
  created_at  timestamptz NOT NULL,
  total       numeric(10,2) NOT NULL
);
INSERT INTO orders (customer_id, status, email, created_at, total)
SELECT (random()*50000)::int + 1,
       (ARRAY['paid','paid','paid','paid','shipped','shipped','shipped','refunded','pending'])[1 + (random()*8)::int],
       'User' || g || '@Example.com',
       timestamptz '2025-01-01' + random() * interval '600 days',
       round((random()*500)::numeric, 2)
FROM generate_series(1, 1000000) g;
VACUUM ANALYZE orders;
```

The heap is 85 MB; the primary-key index 21 MB. Plans below use `EXPLAIN (ANALYZE, BUFFERS,
COSTS OFF)` on PostgreSQL 18.6 — lesson 6 explains how to read them in detail. Timings are from one
run on a laptop with everything cached; compare their *magnitude*, not their exact values.

## From sequential scan to index scan

Without an index on `customer_id`:

```text
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF) SELECT * FROM orders WHERE customer_id = 4242;
 Gather (actual time=2.196..19.732 rows=16.00 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   Buffers: shared hit=10889
   ->  Parallel Seq Scan on orders (actual time=2.020..13.531 rows=5.33 loops=3)
         Filter: (customer_id = 4242)
         Rows Removed by Filter: 333328
 Execution Time: 19.743 ms
```

Three processes read all 10,889 pages to find 16 rows. Add the index:

```sql
CREATE INDEX orders_customer_idx ON orders (customer_id);
```

```text
 Bitmap Heap Scan on orders (actual time=0.026..0.044 rows=16.00 loops=1)
   Recheck Cond: (customer_id = 4242)
   Heap Blocks: exact=16
   Buffers: shared hit=16 read=3
   ->  Bitmap Index Scan on orders_customer_idx (actual time=0.019..0.019 rows=16.00 loops=1)
         Index Cond: (customer_id = 4242)
         Index Searches: 1
         Buffers: shared read=3
 Execution Time: 0.061 ms
```

3 index pages + 16 heap pages instead of 10,889. The B-tree here has two levels above the leaves:

```text
SELECT root, level FROM bt_metap('orders_pkey');
 root | level
------+-------
  412 |     2
```

Level 2 means root → internal page → leaf: three page reads to find any key among a million. B-trees
are wide and shallow — each 8 kB page holds hundreds of keys — so even billions of rows need only
four or five levels.

### Three ways to use an index

- **Index Scan** — walk the index in order and fetch each matching heap row immediately. Best for
  few rows, or when the index order is needed (ORDER BY ... LIMIT).
- **Bitmap Index Scan + Bitmap Heap Scan** — collect all matching row locations into a bitmap,
  sort it by page, then read each heap page once. Best for tens to thousands of rows scattered
  across the table. It loses index order, and can combine several indexes with BitmapAnd/BitmapOr.
- **Index Only Scan** — answer the query from the index alone, without visiting the heap (below).

The planner picks among them, and a plain sequential scan, based on estimated row counts (lesson 7).
A query that matches 30% of a table is often *faster* as a sequential scan; an index is not always
better.

## Column order in multicolumn indexes

A B-tree on `(status, created_at)` is sorted by status first, then by created_at within each status
— like a phone book sorted by surname, then first name.

```sql
CREATE INDEX orders_status_created_idx ON orders (status, created_at);
```

Equality on the first column plus a range on the second is the ideal case — one contiguous slice of
the index:

```text
EXPLAIN ... SELECT count(*) FROM orders WHERE status = 'refunded' AND created_at >= '2026-06-01';
 Aggregate (actual time=1.943..1.943 rows=1.00 loops=1)
   ->  Index Only Scan using orders_status_created_idx on orders (actual time=0.046..1.019 rows=17131.00 loops=1)
         Index Cond: ((status = 'refunded'::text) AND (created_at >= '2026-06-01 00:00:00+05:30'::timestamp with time zone))
         Heap Fetches: 0
         Index Searches: 1
```

Before PostgreSQL 18, a query on **only the second column** could not use this index efficiently.
PostgreSQL 18 added **skip scan**: when the leading column has few distinct values, it jumps to each
one and searches within it:

```text
EXPLAIN ... SELECT id FROM orders WHERE created_at >= '2026-08-20' AND created_at < '2026-08-21';
 Bitmap Heap Scan on orders (actual time=0.343..2.054 rows=1618.00 loops=1)
   ->  Bitmap Index Scan on orders_status_created_idx (actual time=0.233..0.233 rows=1618.00 loops=1)
         Index Cond: ((created_at >= ...) AND (created_at < ...))
         Index Searches: 9
```

`Index Searches: 9` shows nine separate descents into the tree instead of one long scan. Skip scan
is a helpful fallback, not a design strategy: it is only cheap when the leading column has a
handful of values. Rules of thumb for column order:

1. Columns compared with **equality** first, then the column used for a **range** or **ORDER BY**.
2. Among equality columns, put the one queries *always* filter on first.
3. One index on `(a, b)` serves queries on `a` alone and on `a AND b` — so you usually do not need a
   separate index on `a`. (Here `orders_customer_idx` would become redundant once the next index
   exists.)

## Indexes that also sort

"The three latest orders for a customer" with only the single-column index fetches all 16 of the
customer's rows and sorts them:

```text
 Limit (actual time=0.039..0.039 rows=3.00 loops=1)
   ->  Sort (actual time=0.038..0.039 rows=3.00 loops=1)
         Sort Key: created_at DESC
         Sort Method: top-N heapsort  Memory: 25kB
         ->  Bitmap Heap Scan on orders (actual time=0.013..0.032 rows=16.00 loops=1)
```

With an index whose order matches the `ORDER BY`:

```sql
CREATE INDEX orders_customer_created_idx ON orders (customer_id, created_at DESC);
```

```text
 Limit (actual time=0.018..0.021 rows=3.00 loops=1)
   Buffers: shared hit=3 read=3
   ->  Index Scan using orders_customer_created_idx on orders (actual time=0.018..0.020 rows=3.00 loops=1)
         Index Cond: (customer_id = 4242)
```

No sort; the scan stops after three rows. For a customer with 10,000 orders the difference is
enormous. (B-trees can be scanned backwards, so `created_at ASC` would also work here; `DESC` in the
index matters for mixed directions like `ORDER BY a ASC, b DESC`.)

## Expression indexes

An index on `email` cannot help `WHERE lower(email) = ...` — the indexed values are not the values
being compared:

```text
 Parallel Seq Scan on orders (actual time=41.856..61.857 rows=0.33 loops=3)
   Filter: (lower(email) = 'user31337@example.com'::text)
 Execution Time: 64.968 ms
```

Index the expression itself:

```sql
CREATE INDEX orders_email_lower_idx ON orders (lower(email));
```

```text
 Bitmap Heap Scan on orders (actual time=0.032..0.032 rows=1.00 loops=1)
   ->  Bitmap Index Scan on orders_email_lower_idx
         Index Cond: (lower(email) = 'user31337@example.com'::text)
 Execution Time: 0.040 ms
```

The query must use the *same expression*. `WHERE email = 'User31337@Example.com'` still scans the
whole table with only this index. The function must be `IMMUTABLE`, and the planner needs statistics
on the expression, which `ANALYZE` gathers after the index is created.

## Partial indexes

If you only ever query pending orders by date, index only those rows:

```sql
CREATE INDEX orders_pending_idx ON orders (created_at) WHERE status = 'pending';
```

```text
            index            |  size
-----------------------------+---------
 orders_email_lower_idx      | 39 MB
 orders_status_created_idx   | 31 MB
 orders_customer_created_idx | 30 MB
 orders_pkey                 | 21 MB
 orders_customer_idx         | 7808 kB
 orders_pending_idx          | 1392 kB
```

1.4 MB instead of ~20 MB, and it is maintained only for pending rows. The planner uses it when the
query's `WHERE` clause implies the index predicate:

```text
EXPLAIN SELECT * FROM orders WHERE status = 'pending' AND created_at < '2025-01-08';
 Bitmap Heap Scan on orders
   ->  Bitmap Index Scan on orders_pending_idx

EXPLAIN SELECT * FROM orders WHERE status = 'paid' AND created_at < '2025-01-08';
 Bitmap Heap Scan on orders
   ->  Bitmap Index Scan on orders_status_created_idx
```

Partial indexes are ideal for queues (`WHERE processed_at IS NULL`), soft deletes
(`WHERE deleted_at IS NULL`) and rare-but-important states. A parameterised query
`WHERE status = $1` can only use the partial index if the plan is made with the actual value — a
gotcha with generic prepared-statement plans.

## Covering indexes and index-only scans

`INCLUDE` adds columns to the leaf entries without making them part of the search key:

```sql
CREATE INDEX orders_customer_incl_idx ON orders (customer_id) INCLUDE (total);
```

```text
EXPLAIN ... SELECT customer_id, sum(total) FROM orders
           WHERE customer_id BETWEEN 1000 AND 1100 GROUP BY customer_id;
 GroupAggregate (actual time=0.052..0.276 rows=101.00 loops=1)
   Buffers: shared hit=1 read=11
   ->  Index Only Scan using orders_customer_incl_idx on orders (actual time=0.041..0.156 rows=2002.00 loops=1)
         Heap Fetches: 0
```

2,002 rows answered from 12 pages without touching the heap. But watch what happens right after
those rows are updated:

```text
UPDATE orders SET total = total + 1 WHERE customer_id BETWEEN 1000 AND 1100;   -- UPDATE 2002

 GroupAggregate (actual time=0.106..4.295 rows=101.00 loops=1)
   Buffers: shared hit=3328
   ->  Index Only Scan using orders_customer_incl_idx on orders (actual time=0.049..4.157 rows=2002.00 loops=1)
         Heap Fetches: 4004
```

**Heap Fetches: 4004.** An index has no visibility information, so for each entry PostgreSQL checks
the **visibility map**: if the heap page is marked all-visible, the index entry can be trusted. The
update made those pages not-all-visible, so every entry — 2,002 old versions and 2,002 new ones —
needed a heap visit, and buffer reads jumped from 12 to 3,328. After `VACUUM orders` set the bits
again:

```text
   ->  Index Only Scan using orders_customer_incl_idx on orders (actual time=0.030..0.122 rows=2002.00 loops=1)
         Heap Fetches: 0
         Buffers: shared hit=19
```

Index-only scans work best on tables that VACUUM keeps up with. That is one more reason the
insert-triggered autovacuum thresholds from lesson 2 exist.

## LIKE 'prefix%' and collations

```text
CREATE INDEX orders_email_idx ON orders (email);
EXPLAIN SELECT id FROM orders WHERE email LIKE 'User3133%';
   ->  Parallel Seq Scan on orders
         Filter: (email ~~ 'User3133%'::text)
```

The index is ignored. This database's collation is `en_US.UTF-8`, whose sort order is linguistic, so
"strings starting with User3133" is not guaranteed to be a contiguous range of it. Use the
`text_pattern_ops` operator class, which compares byte by byte:

```text
CREATE INDEX orders_email_pattern_idx ON orders (email text_pattern_ops);
 Index Scan using orders_email_pattern_idx on orders
   Index Cond: ((email ~>=~ 'User3133'::text) AND (email ~<~ 'User3134'::text))
   Filter: (email ~~ 'User3133%'::text)
```

The planner turned the prefix into a range. Databases using the `C` collation do not need this.
Patterns with a leading wildcard (`'%3133'`) cannot use a B-tree at all — that is a job for trigram
indexes (next lesson).

## The cost side

Six indexes now take more space than the table (about 130 MB against 85 MB of heap), and every
non-HOT update or insert must maintain all of them. Find indexes that never get used:

```sql
SELECT s.indexrelid::regclass AS index, s.idx_scan, pg_size_pretty(pg_relation_size(s.indexrelid)) AS size
FROM pg_stat_user_indexes s
JOIN pg_index i USING (indexrelid)
WHERE NOT i.indisunique AND s.idx_scan = 0
ORDER BY pg_relation_size(s.indexrelid) DESC;
```

Check the counter has been accumulating for a representative period (and on replicas, which keep
their own statistics) before dropping anything.

## How It Actually Works

PostgreSQL's B-tree is a Lehman–Yao **B+tree**: all keys live in leaf pages, internal pages hold
separator keys and child pointers, and every page has a right-link to its sibling, so concurrent
page splits never block readers — a reader that lands on a page that just split simply follows the
right-link. Leaf entries are `(key, ctid)` pairs kept in key order, with the heap TID as a final
tiebreaker since PostgreSQL 12, so duplicates are ordered too.

Since PostgreSQL 13, leaf pages **deduplicate** equal keys into a single posting-list tuple holding
many TIDs, which is why `orders_customer_idx` (about 20 rows per key) is only 7.8 MB while the
unique primary key on the same number of rows is 21 MB. Since PostgreSQL 14, B-trees also delete
index entries for dead row versions eagerly ("bottom-up deletion") when a page is about to split
because of version churn, which limits index bloat from non-HOT updates.

An index scan descends from the root by binary search on each page, then walks right along the leaf
level until the scan key no longer matches. Skip scan, new in 18, repeats that descent once per
distinct value of an omitted leading column, generating the next value to look for from what it
finds in the index.

## Common mistakes

- Indexing each column separately instead of building one well-ordered multicolumn index.
- Range column first, equality column second: `(created_at, status)` for
  `WHERE status = ? AND created_at > ?`.
- Wrapping indexed columns in functions or casts in the `WHERE` clause.
- Expecting `LIKE 'abc%'` to use an index in a non-C collation without `text_pattern_ops`.
- Adding indexes without measuring write cost or checking for redundancy and HOT impact.
- Expecting index-only scans on tables VACUUM rarely visits.

## Exercise

1. Build the `orders` table. Write the query "total spent per customer in the last 30 days for
   customers 1–500" and find the smallest index that makes it an index-only scan.
2. Create `(created_at, status)` and `(status, created_at)` indexes and compare buffers for
   `WHERE status = 'pending' AND created_at > now() - interval '7 days'`.
3. Remove the redundancy among the indexes in this lesson: which can be dropped without any query
   above getting slower? Verify with `EXPLAIN`.
4. Use `bt_page_stats` and `bt_page_items` from `pageinspect` to look at one leaf page of
   `orders_customer_idx` and find a posting-list tuple.
