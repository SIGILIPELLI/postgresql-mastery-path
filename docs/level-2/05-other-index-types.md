---
description: "When B-trees are wrong: GIN for arrays, JSONB and trigram search, GiST for ranges and nearest-neighbour, BRIN for huge ordered tables, hash."
---

# 05 · GIN, GiST, BRIN & Hash Indexes

A B-tree answers "which rows have a value equal to, or in the range of, X?" Some questions do not
fit that shape:

- "Which articles have the tag `featured`?" — the value is an array; you are asking about one
  element inside it.
- "Which titles contain `ber 4242 ab` anywhere?" — no ordering helps with a substring.
- "Which three shops are nearest to this point?" — distance is not a sort order you can store.
- "Which rows from 1 March?" in a 2-billion-row log table, where a B-tree would itself be hundreds of
  gigabytes.

PostgreSQL ships six index access methods, and extensions add more:

```text
SELECT amname FROM pg_am WHERE amtype = 'i' ORDER BY 1;
 brin
 btree
 gin
 gist
 hash
 spgist
```

This lesson shows when each one earns its place. Everything ran in the `perf` database on
PostgreSQL 18.6.

## GIN: many keys per row

A **G**eneralized **In**verted index works like the index at the back of a book: for every *element*
(a word, an array element, a JSON key/value), it stores the list of rows containing it. Use it when
one row contains many searchable items.

### Arrays

```sql
CREATE TABLE articles (id int PRIMARY KEY, title text, tags text[]);
-- 300,000 rows; 1 in 1,000 is tagged 'featured'
```

Without an index, a containment query reads every page:

```text
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM articles WHERE tags @> ARRAY['featured'];
   ->  Parallel Seq Scan on articles (actual time=0.202..13.170 rows=100.00 loops=3)
         Filter: (tags @> '{featured}'::text[])
         Rows Removed by Filter: 99900
         Buffers: shared hit=4455
 Execution Time: 16.949 ms
```

```sql
CREATE INDEX articles_tags_gin ON articles USING gin (tags);
```

```text
   ->  Bitmap Heap Scan on articles (actual time=0.052..0.279 rows=300.00 loops=1)
         Recheck Cond: (tags @> '{featured}'::text[])
         ->  Bitmap Index Scan on articles_tags_gin (actual time=0.028..0.029 rows=300.00 loops=1)
               Index Cond: (tags @> '{featured}'::text[])
               Buffers: shared hit=2
 Execution Time: 0.302 ms
```

Two index pages to find all 300 rows. GIN supports `@>` (contains), `<@` (is contained by), `&&`
(overlaps) and `=` on arrays. It does **not** help `'featured' = ANY(tags)` — rewrite that as
`tags @> ARRAY['featured']`.

### Substring search with trigrams

`LIKE '%...%'` and `ILIKE` cannot use a B-tree. The `pg_trgm` extension breaks text into
three-character pieces:

```text
SELECT show_trgm('Vacuum');
 {"  v"," va",acu,cuu,"um ",uum,vac}
```

A GIN index over those trigrams can find candidate rows that contain all the trigrams of the search
string:

```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX articles_title_trgm ON articles USING gin (title gin_trgm_ops);
```

```text
EXPLAIN ANALYZE SELECT id FROM articles WHERE title ILIKE '%ber 4242 ab%';
-- before: Parallel Seq Scan ... Execution Time: 38.938 ms
-- after:
 Bitmap Heap Scan on articles (actual time=4.254..4.254 rows=1.00 loops=1)
   Recheck Cond: (title ~~* '%ber 4242 ab%'::text)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on articles_title_trgm (actual time=4.248..4.248 rows=1.00 loops=1)
         Buffers: shared hit=344
 Execution Time: 4.264 ms
```

The index read 344 pages here because common trigrams such as `" ab"` and `ber` appear in almost
every title, so their posting lists are long; rarer search strings are much faster. Trigram indexes
also power similarity search (`title % 'postgress'`, typo-tolerant) and regular-expression matches.
Search strings shorter than three characters cannot use them.

### JSONB

GIN is also the index for `jsonb` containment (`data @> '{"status": "active"}'`) and key-existence
queries. Level 3 · 01 covers the two operator classes and when each fits.

### GIN's costs

An insert into a row with 20 tags means 20 index insertions. To keep that tolerable, GIN buffers new
entries in a **pending list** (`fastupdate`, on by default) and merges them in bulk during VACUUM or
when the list exceeds `gin_pending_list_limit`. The price is that searches must also scan the
unsorted pending list, and an unlucky insert can pay for the merge. On write-heavy tables watch for
latency spikes and consider tuning or disabling `fastupdate`.

## GiST: trees for things that overlap

**G**eneralized **Se**arch **T**ree indexes organise values by bounding regions — a parent entry
"covers" everything below it. That fits data where overlap, containment and distance matter:
ranges, geometric types, PostGIS geometries, full-text vectors and, via `btree_gist`, scalars.

You already used one: the exclusion constraint in Level 1 · 06 is enforced by a GiST index that finds
overlapping ranges.

GiST also supports **nearest-neighbour ordering** — "ORDER BY distance LIMIT n" without computing
the distance to every row:

```sql
CREATE TABLE shops (id int PRIMARY KEY, name text, location point);
-- 200,000 random points in a 1000 x 1000 square
CREATE INDEX shops_location_gist ON shops USING gist (location);
```

```text
EXPLAIN (ANALYZE, BUFFERS) SELECT id, location <-> point(500,500) AS dist
FROM shops ORDER BY location <-> point(500,500) LIMIT 3;
 Limit (actual time=0.371..0.374 rows=3.00 loops=1)
   Buffers: shared hit=3 read=3
   ->  Index Scan using shops_location_gist on shops (actual time=0.371..0.373 rows=3.00 loops=1)
         Order By: (location <-> '(500,500)'::point)
```

Six pages to find the three nearest of 200,000 points. The same pattern finds the closest trigram
matches (`ORDER BY title <-> 'search'` with a GiST trigram index) and, with pgvector, nearest
embeddings — though that extension also brings its own index types.

**GIN or GiST for the same data?** For trigram and full-text search both exist. GIN is generally
faster to search and slower to update; GiST is smaller, faster to update, can be lossy (rechecks)
and supports distance ordering.

## SP-GiST

Space-partitioned GiST builds unbalanced structures — quadtrees, k-d trees, radix trees. It suits
data with natural non-overlapping partitions: points, IP addresses (`inet`), text prefixes. It is
less commonly chosen by hand; when a type supports it, benchmark it against GiST.

## BRIN: tiny indexes for naturally ordered data

A **B**lock **R**ange **IN**dex stores only a summary — min and max — for each range of table pages
(128 pages by default). It is useful only when the column's values follow the physical order of the
table, as timestamps do in an append-only log.

```sql
CREATE TABLE readings (id bigint GENERATED ALWAYS AS IDENTITY, taken_at timestamptz NOT NULL,
                       sensor int, value float8);
-- 2,000,000 rows, one every 10 seconds, inserted in time order
CREATE INDEX readings_taken_btree ON readings (taken_at);
CREATE INDEX readings_taken_brin  ON readings USING brin (taken_at);
```

```text
        index         | size
----------------------+-------
 readings_taken_btree | 43 MB
 readings_taken_brin  | 24 kB
```

The table is 115 MB. The BRIN index is about 1/1800th the size of the B-tree. Querying one day:

```text
 Bitmap Heap Scan on readings (actual time=0.225..0.826 rows=8640.00 loops=1)
   Rows Removed by Index Recheck: 8768
   Heap Blocks: lossy=128
   Buffers: shared hit=133
   ->  Bitmap Index Scan on readings_taken_brin (actual time=0.021..0.021 rows=1280.00 loops=1)
         Buffers: shared hit=5
 Execution Time: 1.164 ms
```

BRIN is **lossy**: it returns whole page ranges that *might* contain matches, and the heap scan
rechecks every row (8,768 rows discarded). Still, 133 pages instead of 14,700. The planner knows when
BRIN can work from the column's physical correlation:

```text
SELECT correlation FROM pg_stats WHERE tablename = 'readings' AND attname = 'taken_at';
 correlation
-------------
           1
```

Now the same data in random physical order:

```text
SELECT correlation FROM pg_stats WHERE tablename = 'readings_shuffled' AND attname = 'taken_at';
 0.008978395

-- forcing the BRIN index (enable_seqscan = off) for the same one-day query:
 Parallel Bitmap Heap Scan on readings_shuffled (actual time=0.171..136.542 rows=2880.00 loops=3)
   Rows Removed by Index Recheck: 663787
   Heap Blocks: lossy=5361
   Buffers: shared hit=46 read=14719 written=8148
 Execution Time: 140.879 ms
```

Every block range contains timestamps from the whole year, so every range "might match" and the
whole table is read. BRIN is a great fit for append-only time-series, logs and event tables, and
useless otherwise. Updates that move rows around and bulk deletes followed by reuse of space erode
the correlation over time. Ranges added after index creation are summarised by VACUUM (or
immediately, if you create the index with `autosummarize = on`).

## Hash indexes

Hash indexes store a 32-bit hash of each value and support only `=`.

```sql
CREATE TABLE sessions (token text PRIMARY KEY, user_id int);
-- 500,000 rows of 64-character tokens
CREATE INDEX sessions_token_hash ON sessions USING hash (token);
```

```text
     indexrelid      | pg_size_pretty
---------------------+----------------
 sessions_pkey       | 60 MB
 sessions_token_hash | 20 MB

 Index Scan using sessions_token_hash on sessions
   Index Cond: (token = '28dd2c7955ce926456240b2ff0100bde5737034557ef5b8c02c0e46513b98f90'::text)
```

On long keys the hash index is much smaller because it stores 4-byte hashes rather than the
64-character values. Since PostgreSQL 10 they are WAL-logged and crash-safe (avoid them on older
versions). They cannot enforce uniqueness, support ranges or sorting, or be multicolumn — so a
B-tree is still the default, and hash is an optimisation for large keys looked up only by equality.

## Choosing

| Question | Index |
|----------|-------|
| equality, ranges, sorting, uniqueness | B-tree |
| array/JSONB contains, key exists, full-text match | GIN |
| `LIKE '%x%'`, similarity, regex | GIN (or GiST) with `pg_trgm` |
| ranges overlap, exclusion constraints, geometry, nearest neighbour | GiST |
| huge append-only table, filter on time or ID | BRIN |
| equality only on long values | hash (or B-tree) |

## How It Actually Works

All index types plug into the same **index access method** interface: functions to build, insert,
scan (returning TIDs one by one or as a bitmap), and vacuum. The planner asks each index's
**operator class** which operators it can accelerate (`pg_amop`) — that is why
`text_pattern_ops` and `gin_trgm_ops` change what an index can do on the same column.

- **GIN** has a B-tree of keys (elements); each key points to a sorted posting list or, when long, a
  posting tree of TIDs. A query extracts keys from the search value, fetches their TID lists, and
  combines them according to the operator (all keys for `@>`, any for `&&`).
- **GiST** is a balanced tree in which each internal entry is a "union" or bounding key of its
  children, defined by the operator class's `union`, `consistent`, `penalty` and `picksplit`
  functions. Search descends every child whose key is *consistent* with the query; nearest-neighbour
  search uses a priority queue ordered by the `distance` function.
- **BRIN** stores one summary tuple per page range in a small revmap-indexed structure. A scan reads
  all summaries (tiny), adds every range whose summary is consistent with the query to a lossy
  bitmap, and the heap scan does the rest.
- **Hash** uses buckets of overflow-chained pages addressed by the hash value, and splits buckets
  incrementally as the index grows.

## Common mistakes

- Writing `x = ANY(array_col)` and expecting a GIN index to be used.
- BRIN on columns that are not physically correlated with insertion order.
- Trigram indexes for searches of one or two characters.
- GIN on extremely write-heavy columns without considering `fastupdate` behaviour.
- Reaching for exotic indexes before checking whether a correctly ordered B-tree solves the problem.

## Exercise

1. Add a GIN index for a `tags text[]` column in your own schema and compare buffers for `@>`,
   `&&` and `= ANY` queries.
2. Create a `pg_trgm` GiST index on a names table and implement "did you mean?" search with
   `ORDER BY name <-> 'jonh' LIMIT 5`.
3. Build a 10-million-row append-only log table, compare B-tree and BRIN for a one-hour query, then
   try BRIN with `pages_per_range = 32` and `= 512`. Which is smallest? Which reads fewest pages?
4. Delete the oldest half of the log, insert new rows (they will reuse the freed space), and check
   what happens to `correlation` and the BRIN query.
