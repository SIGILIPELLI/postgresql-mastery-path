---
description: "JSONB done properly: operators, containment, jsonpath, JSON_TABLE, jsonb_set updates, GIN operator classes and when to use real columns."
---

# 01 · JSONB in Depth

PostgreSQL's `jsonb` type lets one database serve both relational and document-shaped data: a
webhook payload, per-customer settings, attributes that differ by product category. Used well, it
removes the need for a separate document store. Used badly, it becomes a schemaless swamp with
slow queries and silent typos.

This lesson covers querying, modifying and indexing `jsonb`, and the decision that matters most:
which data belongs in JSON and which in columns. Outputs are from PostgreSQL 18.6.

## json vs jsonb

```text
SELECT '{"b": 1, "a": 2, "a": 3}'::json AS as_json, '{"b": 1, "a": 2, "a": 3}'::jsonb AS as_jsonb;
         as_json          |     as_jsonb
--------------------------+------------------
 {"b": 1, "a": 2, "a": 3} | {"a": 3, "b": 1}
```

`json` stores the text exactly as given and re-parses it on every access. `jsonb` parses once into a
binary format: keys are sorted and de-duplicated (last one wins), whitespace is dropped, and
operations do not need to re-parse. Use `jsonb` unless you must preserve the exact input text — for
example, to verify a signature over a webhook body.

## Reading values

```sql
CREATE TABLE events (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, data jsonb NOT NULL);
INSERT INTO events (data) VALUES
 ('{"type": "signup", "user": {"id": 7, "country": "IN"}, "plan": "pro", "tags": ["web", "promo"]}'),
 ('{"type": "purchase", "user": {"id": 7, "country": "IN"}, "amount": 49.5,
    "items": [{"sku": "A1", "qty": 2}, {"sku": "B2", "qty": 1}]}'),
 ('{"type": "purchase", "user": {"id": 9, "country": "DE"}, "amount": 12, "items": [{"sku": "B2", "qty": 5}]}');

SELECT data->'user' AS user_json, data->'user'->>'country' AS country,
       data #>> '{items,0,sku}' AS first_sku,
       pg_typeof(data->'amount') AS t1, pg_typeof(data->>'amount') AS t2
FROM events ORDER BY id;
```

```text
         user_json          | country | first_sku |  t1   |  t2
----------------------------+---------+-----------+-------+------
 {"id": 7, "country": "IN"} | IN      |           | jsonb | text
 {"id": 7, "country": "IN"} | IN      | A1        | jsonb | text
 {"id": 9, "country": "DE"} | DE      | B2        | jsonb | text
```

| Operator | Returns |
|----------|---------|
| `->  'key'` / `-> 2` | `jsonb` value of a key / array element |
| `->> 'key'` | that value as `text` |
| `#> '{a,b,0}'` / `#>> '{a,b,0}'` | value at a path, as `jsonb` / `text` |
| `@>` | contains (left contains right) |
| `?`, `?|`, `?&` | key exists / any of / all of |
| `@?`, `@@` | jsonpath exists / jsonpath predicate |

A missing key is `NULL`, not an error — so a typo like `data->>'contry'` silently returns NULL for
every row. That is the main hazard of JSON storage.

`->>` returns **text**. To do arithmetic or numeric comparison, cast: `(data->>'amount')::numeric`.
Comparing text numbers sorts `'100'` before `'9'`.

## Containment: the query GIN indexes love

```sql
SELECT id FROM events WHERE data @> '{"type": "purchase", "user": {"country": "IN"}}';
--  id = 2
```

`@>` asks "does the document contain this structure?" It matches nested objects and array elements,
and it is the operator best supported by GIN indexes (below). Prefer it to chains of
`data->'user'->>'country' = 'IN' AND ...` when filtering by fixed values.

## jsonpath

SQL/JSON path expressions (PostgreSQL 12+) navigate and filter inside a document:

```sql
SELECT id, jsonb_path_query_array(data, '$.items[*] ? (@.qty > 1).sku') AS bulk_skus
FROM events
WHERE data @? '$.items[*] ? (@.qty > 1)';
```

```text
 id | bulk_skus
----+-----------
  2 | ["A1"]
  3 | ["B2"]
```

`$` is the document, `[*]` every array element, `? (...)` a filter with `@` as the current item. The
`@?` operator ("does the path return anything?") can use a GIN index.

## Turning JSON into rows

Arrays of objects become rows with `jsonb_to_recordset`:

```sql
SELECT e.id, i.sku, i.qty
FROM events e, jsonb_to_recordset(e.data->'items') AS i(sku text, qty int)
ORDER BY 1, 2;
```

```text
 id | sku | qty
----+-----+-----
  2 | A1  |   2
  2 | B2  |   1
  3 | B2  |   5
```

PostgreSQL 17 added the SQL-standard `JSON_TABLE`, which maps paths to typed columns with defaults:

```sql
SELECT e.id, jt.*
FROM events e,
     JSON_TABLE(e.data, '$' COLUMNS (
       kind    text    PATH '$.type',
       country text    PATH '$.user.country',
       amount  numeric PATH '$.amount' DEFAULT 0 ON EMPTY)) AS jt
ORDER BY e.id;
```

```text
 id |   kind   | country | amount
----+----------+---------+--------
  1 | signup   | IN      |      0
  2 | purchase | IN      |   49.5
  3 | purchase | FR      |      0
```

(Row 3's country and amount reflect the updates in the next section, which ran first.)

## Modifying documents

```sql
UPDATE events SET data = jsonb_set(data, '{user,country}', '"FR"') WHERE id = 3
RETURNING data->'user';
-- {"id": 9, "country": "FR"}
```

`jsonb_set(target, path, new_value)` — note the new value is itself JSON, so a string needs inner
quotes. `||` merges objects (shallowly) and `-` removes a key. A real gotcha I hit writing this
lesson:

```text
UPDATE events SET data = data || '{"refunded": true}' - 'amount' WHERE id = 3;
ERROR:  operator is not unique: unknown - unknown
HINT:  Could not choose a best candidate operator. You might need to add explicit type casts.
```

`-` binds more tightly than `||`, so PostgreSQL tried to evaluate `'{"refunded": true}' - 'amount'`
between two untyped literals first. Parenthesise:

```sql
UPDATE events SET data = (data || '{"refunded": true}') - 'amount' WHERE id = 3 RETURNING data;
```

```text
 {"type": "purchase", "user": {"id": 9, "country": "FR"}, "items": [{"qty": 5, "sku": "B2"}], "refunded": true}
```

### Every update rewrites the whole document

`jsonb` has no in-place partial update. A table with a 170 kB document (116 kB after TOAST
compression) and a separate timestamp column:

```text
 jsonb_key_update | plain_column_update_1 | plain_column_update_2
------------------+-----------------------+-----------------------
 252 kB           | 208 bytes             | 208 bytes
```

Changing one small key inside the document produced 252 kB of WAL (the whole new TOAST value, plus
full-page images because it was the first change after a checkpoint). Updating the ordinary column on
the same row produced 208 bytes, because the unchanged TOASTed document is not copied (Level 2 · 03).
Keep frequently changing values — counters, `last_seen_at`, status — in columns, not inside a big
document.

## Indexing jsonb

Test data: 500,000 events, 83 MB of heap.

```sql
CREATE INDEX big_events_gin      ON big_events USING gin (data);                  -- jsonb_ops
CREATE INDEX big_events_gin_path ON big_events USING gin (data jsonb_path_ops);
CREATE INDEX big_events_type     ON big_events ((data->>'type'));                 -- B-tree on one key
```

```text
        index        |  size
---------------------+---------
 big_events_pkey     | 11 MB
 big_events_gin      | 61 MB
 big_events_gin_path | 35 MB
 big_events_type     | 3424 kB
```

The two GIN operator classes differ in what they store and support:

| | `jsonb_ops` (default) | `jsonb_path_ops` |
|---|---|---|
| indexes | every key and every value separately | a hash of each full path-to-value |
| supports | `@>`, `?`, `?|`, `?&`, `@?`, `@@` | `@>`, `@?`, `@@` only |
| size here | 61 MB | 35 MB |

With only `jsonb_path_ops`:

```text
EXPLAIN (ANALYZE) SELECT count(*) FROM big_events
WHERE data @> '{"plan": "enterprise", "user": {"country": "IN"}}';
   ->  Bitmap Heap Scan on big_events (actual time=0.648..8.432 rows=1666.00 loops=1)
         ->  Bitmap Index Scan on big_events_gin_path

EXPLAIN SELECT count(*) FROM big_events WHERE data @? '$.user ? (@.country == "IN")';
         ->  Bitmap Index Scan on big_events_gin_path

EXPLAIN SELECT count(*) FROM big_events WHERE data ? 'refunded';
               ->  Parallel Seq Scan on big_events          -- key-exists needs jsonb_ops
```

And neither GIN class helps the most common style of query:

```text
EXPLAIN SELECT count(*) FROM big_events WHERE data->>'plan' = 'enterprise';
               ->  Parallel Seq Scan on big_events
                     Filter: ((data ->> 'plan'::text) = 'enterprise'::text)
```

GIN indexes operators, not expressions. Either rewrite as `data @> '{"plan": "enterprise"}'`, or —
when one key is queried constantly — use a B-tree expression index, which is tiny and also supports
ranges and sorting:

```text
EXPLAIN SELECT count(*) FROM big_events WHERE data->>'type' = 'signup';
         ->  Bitmap Index Scan on big_events_type
               Index Cond: ((data ->> 'type'::text) = 'signup'::text)
```

## Columns or JSON?

A rule of thumb that holds up well:

- **Columns** for anything you filter, join, sort or aggregate on regularly; anything with
  constraints (NOT NULL, foreign keys, ranges); anything updated often.
- **JSON** for data that is genuinely variable in shape, read and written as a whole, or stored for
  reference (raw webhook payloads, third-party API responses, user preferences).

A common hybrid: keep the raw payload in `jsonb`, and promote the fields you query into **generated
columns** so they are typed, indexable and impossible to drift:

```sql
ALTER TABLE big_events
  ADD COLUMN event_type text GENERATED ALWAYS AS (data->>'type') STORED;
CREATE INDEX ON big_events (event_type);
```

You can also validate shape with CHECK constraints, e.g.
`CHECK (jsonb_typeof(data->'items') = 'array')` or `CHECK (data ? 'type')`.

## How It Actually Works

A `jsonb` value is stored as a tree in a binary format: a header with the count of children and a
flag for object/array/scalar, then an array of "JEntry" words giving each child's type and offset,
then the keys (sorted, for objects) and values. Because object keys are sorted and their offsets are
known, finding a key is a binary search inside the value, not a scan of text. Numbers are stored as
`numeric`, which is why `49.5` round-trips exactly. The whole value is a single varlena, so it is
compressed and TOASTed as a unit, and any change produces a complete new value.

`jsonb_ops` GIN extracts one index key per key name and one per scalar value (flagged to distinguish
them), so a containment query becomes "rows having all these keys and values" — then rechecks
candidates against the real structure, since key `country` and value `IN` might come from different
places. `jsonb_path_ops` instead hashes each path from root to scalar (`user.country = "IN"`) into a
single index key, which is more selective and smaller but cannot answer "does key X exist" on its own.

## Common mistakes

- Typos in key names silently returning NULL.
- Comparing `->>` results as text when they hold numbers or dates.
- Expecting a GIN index to speed up `data->>'k' = 'v'`.
- Storing counters or timestamps that change constantly inside large documents.
- Using JSON for relationships that need foreign keys.
- Operator-precedence surprises when mixing `||` and `-`.

## Exercise

1. Store a sample of real webhook payloads (any public API's example JSON works) in a `jsonb` column.
   Write queries using `->>`, `@>`, a jsonpath filter and `JSON_TABLE`.
2. Promote two frequently queried fields into stored generated columns and index them. Compare
   plans with the equivalent `@>` query on a `jsonb_path_ops` GIN index.
3. Add CHECK constraints that require a `type` key and that `items`, if present, is an array of
   objects each having a numeric `qty`. (Hint: `jsonb_path_exists` with a filter, or `NOT EXISTS`
   over `jsonb_array_elements`.)
4. Measure WAL for updating one key in a 50 kB document versus an adjacent integer column, as above.
