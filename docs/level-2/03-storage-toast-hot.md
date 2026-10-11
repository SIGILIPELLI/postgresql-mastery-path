---
description: "How PostgreSQL lays rows out on 8 kB pages, when TOAST compresses or moves values out of line, and how fillfactor enables cheap HOT updates."
---

# 03 · Pages, TOAST, Fillfactor & HOT Updates

The last two lessons established that rows are versioned and that VACUUM cleans up. This lesson
zooms into the storage layer, because three of its mechanisms decide how expensive your writes are:

- **Pages** — the 8 kB unit of everything.
- **TOAST** — how a 1 MB text value fits in an 8 kB page.
- **HOT updates** — the optimisation that lets an UPDATE skip touching indexes, and the
  `fillfactor` setting that makes it possible.

Output is from PostgreSQL 18.6 with the `pageinspect` and `pgstattuple` extensions.

## Anatomy of a page

```sql
SELECT * FROM page_header(get_raw_page('mv', 0));
```

```text
    lsn    | checksum | flags | lower | upper | special | pagesize | version | prune_xid
-----------+----------+-------+-------+-------+---------+----------+---------+-----------
 0/6544400 |        0 |     0 |    40 |  8032 |    8192 |     8192 |       4 |    101333
```

- **lsn** — the WAL position of the last change to this page. Recovery uses it to decide whether a
  WAL record still needs replaying on this page (lesson 9).
- **lower / upper** — the free space is between them. Line pointers grow up from `lower`; tuple
  data grows down from `upper`. Here 40 − 24 header bytes = 16 bytes of line pointers (four
  4-byte pointers), and the tuples occupy 8032–8192.
- **prune_xid** — a hint that this page contains tuples deleted by transaction 101333 or later, so
  a future reader may be able to prune it.
- **checksum** — computed when the page is written out, so a page still in shared buffers can show 0.

The maximum row size that fits on a page is roughly 8 kB minus overheads. Larger values need TOAST.

## TOAST: The Oversized-Attribute Storage Technique

When a row would exceed about 2 kB (`TOAST_TUPLE_THRESHOLD`), PostgreSQL tries, column by column:

1. **compress** large variable-length values inline;
2. if still too big, **move** them out of line into the table's TOAST table, in ~2 kB chunks,
   leaving an 18-byte pointer in the row.

```sql
CREATE TABLE docs (id int PRIMARY KEY, body text);
INSERT INTO docs VALUES
  (1, repeat('short ', 10)),
  (2, repeat('compressible text ', 1000)),
  (3, (SELECT string_agg(md5(g::text), '') FROM generate_series(1, 500) g));

SELECT id, length(body) AS chars, pg_column_size(body) AS stored_bytes,
       pg_column_compression(body) AS method
FROM docs ORDER BY id;
```

```text
 id | chars | stored_bytes | method
----+-------+--------------+--------
  1 |    60 |           61 |
  2 | 18000 |          235 | pglz
  3 | 16000 |        16000 |
```

- Row 1 is small and stored as-is.
- Row 2 is 18,000 characters of repetition, compressed to 235 bytes with `pglz`, small enough to
  stay inline.
- Row 3 is 16,000 hex characters from MD5 hashes. `pglz` gave up on it (it abandons compression that
  does not save enough), so the full 16,000 bytes were moved to the TOAST table:

```sql
SELECT reltoastrelid::regclass FROM pg_class WHERE relname = 'docs';   -- pg_toast.pg_toast_29490
SELECT pg_size_pretty(pg_relation_size('docs')) AS heap,
       pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast
FROM pg_class WHERE relname = 'docs';
```

```text
    heap    | toast
------------+-------
 8192 bytes | 24 kB
```

### Choosing the compression method

`pglz` is the default; `lz4` is usually faster and, on this data, smaller:

```sql
ALTER TABLE docs ALTER COLUMN body SET COMPRESSION lz4;
INSERT INTO docs VALUES (4, repeat('compressible text ', 1000));
SELECT id, pg_column_size(body), pg_column_compression(body) FROM docs WHERE id IN (2, 4);
```

```text
 id | stored_bytes | method
----+--------------+--------
  2 |          235 | pglz
  4 |          107 | lz4
```

Changing the setting affects only newly written values — row 2 kept `pglz`. `lz4` requires a server
built with LZ4 support (most packages are); set `default_toast_compression = lz4` to make it the
default.

### Per-column storage strategies

`ALTER TABLE t ALTER COLUMN c SET STORAGE ...` chooses among `PLAIN` (never TOAST — fixed-width
types), `MAIN` (compress, move out only as a last resort), `EXTERNAL` (move out, never compress —
good for already-compressed data like JPEG bytes, and makes `substring()` on large text fast because
only the needed chunks are fetched) and `EXTENDED` (compress then move out; the default for most
variable-length types).

### TOAST consequences

- `SELECT *` on a table with large TOASTed columns reads the TOAST table for every row. Selecting
  only the columns you need can be dramatically faster.
- Updating a row **without** changing its TOASTed column does not copy the TOASTed value — the new
  row version reuses the same pointer. Updating the big column writes a whole new TOAST value.
- `jsonb` documents are TOASTed as a single value. Changing one key in a 100 kB document rewrites
  100 kB (Level 3 · 01).

## HOT updates

From lesson 1: an UPDATE creates a new row version with a new `ctid`. Every index on the table points
at `ctid`s. So, naively, every UPDATE must insert a new entry into **every index** — even indexes on
columns that did not change. On a table with eight indexes, incrementing a counter means nine writes.

**Heap-Only Tuple (HOT)** updates avoid that when two conditions hold:

1. **No indexed column changed** (summarising indexes like BRIN excepted).
2. **The new version fits on the same page** as the old one.

Then the new version is written on the same page, the old version's header points to it, and the
indexes keep pointing at the *old* line pointer. An index lookup lands on the old slot and follows the
chain within the page. No index writes at all.

Condition 2 is where `fillfactor` comes in. By default tables are packed 100% full, so there is
rarely room on the same page. Two identical tables, one with `fillfactor = 70` (leave 30% of each
page free on insert):

```sql
CREATE TABLE counters_ff100 (id int PRIMARY KEY, name text, hits int NOT NULL DEFAULT 0)
  WITH (autovacuum_enabled = false);
CREATE TABLE counters_ff70  (id int PRIMARY KEY, name text, hits int NOT NULL DEFAULT 0)
  WITH (fillfactor = 70, autovacuum_enabled = false);
CREATE INDEX ON counters_ff100 (name);
CREATE INDEX ON counters_ff70 (name);
-- 50,000 rows each, then: UPDATE ... SET hits = hits + 1 on every row
```

After one full-table update:

```text
    relname     | n_tup_upd | n_tup_hot_upd | hot_pct
----------------+-----------+---------------+---------
 counters_ff100 |     50000 |             0 |       0
 counters_ff70  |     50000 |         22038 |      44
```

Then I reset the counters and ran two more rounds of *vacuum, update everything*:

```text
    relname     | n_tup_upd | n_tup_hot_upd | hot_pct
----------------+-----------+---------------+---------
 counters_ff100 |    100000 |           148 |       0
 counters_ff70  |    100000 |         78095 |      78
```

The packed table essentially never gets HOT updates; the one with headroom gets them for most rows.
Sizes after all rounds:

```text
    relname     |  heap   | indexes
----------------+---------+---------
 counters_ff100 | 5096 kB | 4992 kB
 counters_ff70  | 6880 kB | 4976 kB
```

The trade-off is visible: the `fillfactor = 70` heap is about a third larger. In this small test
the indexes did not grow differently, because VACUUM between rounds let the index pages be reused;
what HOT saved was index *write work* (two index insertions avoided per HOT update here) and the
later index cleanup that VACUUM would otherwise have to do.

Updating an indexed column defeats HOT regardless of fillfactor:

```sql
UPDATE counters_ff70 SET name = name || '-v2' WHERE id <= 1000;
-- n_tup_upd 50000 → 51000, n_tup_hot_upd unchanged at 22038 (counters before the reset)
```

### Using this in practice

- Check `n_tup_hot_upd / n_tup_upd` in `pg_stat_user_tables` for your most-updated tables.
- For tables with frequent updates to non-indexed columns (counters, statuses, `last_seen_at`),
  lower `fillfactor` to 70–90: `ALTER TABLE t SET (fillfactor = 80);` then rewrite it (or let new
  pages fill at the new setting).
- **Every index you add can turn HOT updates into non-HOT updates.** An index on `updated_at` on a
  table where every update sets `updated_at` disables HOT completely for that table. That cost is
  invisible in a read benchmark and very visible in write throughput.

## How It Actually Works

When `heap_update` runs, it compares the old and new tuples' values for every column covered by a
non-summarising index. If none changed and the target page has enough free space, it writes the new
version on the same page, sets `HEAP_HOT_UPDATED` on the old tuple and `HEAP_ONLY_TUPLE` on the new
one, and skips index insertion entirely. A chain of HOT versions on one page is reachable only
through the root line pointer the indexes know about.

Later, any backend that reads the page and finds that old chain members are dead to everyone can
**prune** the page on the spot — no VACUUM needed. Pruning removes the dead versions, compacts the
page, and turns the root line pointer into a *redirect* to the live version, keeping the indexes
valid. That is why HOT plus fillfactor produces a steady state: space freed by pruning on a page is
reused by the next update on the same page.

TOAST works at the tuple-forming stage: if the tuple is larger than the threshold, the toaster loops
over the widest variable-length columns, compressing first and then moving values out of line, until
the row fits under `TOAST_TUPLE_TARGET`. Out-of-line values are stored as chunks in
`pg_toast.pg_toast_<oid>`, keyed by a value ID and chunk sequence number with its own index, and
read back ("detoasted") only when a query actually needs that column's value.

## Common mistakes

- Indexing a column that changes on every update (often `updated_at`), killing HOT.
- Leaving `fillfactor` at 100 on small, extremely hot tables like counters or job queues.
- Lowering `fillfactor` on append-only tables, where it only wastes space.
- `SELECT *` from tables with large TOASTed columns when you need two small columns.
- Storing big binary blobs in heavily updated rows, so every update risks rewriting them.
- Expecting `ALTER ... SET COMPRESSION` to recompress existing data.

## Exercise

1. Create a table with a `text` column and insert values of 100 bytes, 3 kB of repetitive text, and
   50 kB of random text (`string_agg(md5(random()::text), '')`). Use `pg_column_size` and
   `pg_column_compression` to classify each, and find the TOAST table's size.
2. Compare `SELECT id FROM t` with `SELECT * FROM t` on 10,000 rows of 50 kB values using
   `\timing`. Explain the difference.
3. Reproduce the fillfactor experiment with your own numbers, then add an index on `hits` and run
   the updates again. What happens to `hot_pct`?
4. Use `heap_page_items` on a page of the `counters_ff70` table after a few HOT updates and identify
   redirect line pointers (`lp_flags = 2`) and heap-only tuples.
