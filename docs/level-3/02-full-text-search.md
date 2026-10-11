---
description: "Search inside PostgreSQL: tsvector and tsquery, stemming, websearch syntax, weighted ranking, snippets, unaccent and GIN indexes."
---

# 02 · Full-Text Search

For a large class of applications — a help centre, product search for a few hundred thousand items,
searching notes or tickets — PostgreSQL's built-in full-text search is enough, and it has a huge
advantage over a separate search engine: the index is transactionally consistent with your data, and
there is nothing else to deploy, sync or monitor.

This lesson builds search over a small knowledge base and then measures it at scale. Outputs are
from PostgreSQL 18.6.

## Documents become lexemes

`LIKE '%run%'` matches characters. Full-text search matches *words*, normalised:

```text
SELECT to_tsvector('english', 'The runners were running quickly through the parks');
 'park':8 'quick':5 'run':4 'runner':2
```

The `english` configuration split the text into tokens, dropped **stop words** ("the", "were",
"through"), and **stemmed** the rest: "running" → `run`, "parks" → `park`, "quickly" → `quick`. The
numbers are word positions, used for phrase search and ranking. Compare the `simple` configuration,
which only lower-cases:

```text
SELECT to_tsvector('simple', 'The runners were running quickly through the parks');
 'parks':8 'quickly':5 'runners':2 'running':4 'the':1,7 'through':6 'were':3
```

`ts_debug` shows what happens to each token and which dictionary handled it:

```text
SELECT * FROM ts_debug('english', 'PostgreSQL 18 runs faster');
   alias   |   description    |   token    |  dictionaries  |  dictionary  |   lexemes
-----------+------------------+------------+----------------+--------------+--------------
 asciiword | Word, all ASCII  | PostgreSQL | {english_stem} | english_stem | {postgresql}
 uint      | Unsigned integer | 18         | {simple}       | simple       | {18}
 asciiword | Word, all ASCII  | runs       | {english_stem} | english_stem | {run}
 asciiword | Word, all ASCII  | faster     | {english_stem} | english_stem | {faster}
```

(Blank-space rows omitted.) Stemming is algorithmic (the Snowball stemmer), so it is not a
dictionary of real words: "freeze" becomes `freez`, and "faster" stays `faster`.

## Queries

A `tsquery` is a boolean expression of lexemes, and `@@` tests a vector against it:

```text
SELECT to_tsvector('english', 'Parks are great for running') @@ to_tsquery('english', 'run & park');
 t
```

Four ways to build queries:

```text
SELECT to_tsquery('english', 'running & park')                                    AS q1,
       plainto_tsquery('english', 'running in parks')                             AS q2,
       websearch_to_tsquery('english', '"vacuum freeze" -wraparound or autovacuum') AS q3;
       q1       |       q2       |                         q3
----------------+----------------+-----------------------------------------------------
 'run' & 'park' | 'run' & 'park' | 'vacuum' <-> 'freez' & !'wraparound' | 'autovacuum'
```

- `to_tsquery` — you write the operators (`&`, `|`, `!`, `<->` for "followed by", `:*` for prefix).
  It raises a syntax error on malformed input, so **never pass raw user input to it**.
- `plainto_tsquery` — ANDs all the words.
- `phraseto_tsquery` — words must be adjacent: `'tune' <-> 'autovacuum'`.
- `websearch_to_tsquery` — accepts search-box syntax: quotes for phrases, `-` for NOT, `or`. It never
  raises a syntax error, which makes it the right choice for user input.

## A searchable table

Store the vector in a **generated column**, so it can never drift from the text, and weight the title
above the body:

```sql
CREATE TABLE kb (
  id     int PRIMARY KEY,
  title  text NOT NULL,
  body   text NOT NULL,
  search tsvector GENERATED ALWAYS AS (
           setweight(to_tsvector('english', title), 'A') ||
           setweight(to_tsvector('english', body),  'B')) STORED
);
INSERT INTO kb (id, title, body) VALUES
 (1, 'Tuning autovacuum for busy tables', 'Lower the scale factor and raise the cost limit so autovacuum keeps up with dead tuples on large, frequently updated tables.'),
 (2, 'Why VACUUM FREEZE matters', 'Freezing old rows prevents transaction ID wraparound. Monitor the age of datfrozenxid and let autovacuum do its work.'),
 (3, 'Choosing an index type', 'B-tree indexes handle equality and ranges. GIN indexes help with arrays, JSONB and full-text search vectors.'),
 (4, 'Replication slots and disk usage', 'An inactive replication slot retains WAL and can fill the disk. Drop slots that are no longer consumed.'),
 (5, 'Connection pooling basics', 'Each connection is a process. A pooler such as PgBouncer lets thousands of clients share a few server connections.');
CREATE INDEX kb_search ON kb USING gin (search);
```

The configuration must be named explicitly (`'english'`) in the generated column, because the
one-argument form depends on the `default_text_search_config` setting and is therefore not immutable.

## Ranking

```sql
SELECT id, title, ts_rank(search, q) AS rank, ts_rank_cd(search, q) AS rank_cd
FROM kb, websearch_to_tsquery('english', 'autovacuum') q
WHERE search @@ q
ORDER BY rank DESC;
```

```text
 id |               title               |    rank    | rank_cd
----+-----------------------------------+------------+---------
  1 | Tuning autovacuum for busy tables | 0.66871977 |     1.4
  2 | Why VACUUM FREEZE matters         | 0.24317084 |     0.4
```

Both mention autovacuum, but article 1 has it in the title (weight A) and twice overall, so it ranks
higher. `ts_rank` weighs term frequency; `ts_rank_cd` ("cover density") also rewards matched terms
appearing close together. Both accept a normalisation flag to penalise long documents. Ranking needs
the vector of every matching row, so it is computed after the index has found matches — rank a few
hundred candidates, not a million.

Note how AND semantics narrows results:

```text
... websearch_to_tsquery('english', 'autovacuum tables') ...
 id |               title               |    rank
----+-----------------------------------+------------
  1 | Tuning autovacuum for busy tables | 0.98200154
```

Article 2 mentions autovacuum but not tables, so it no longer matches.

## Snippets and prefix search

```sql
SELECT id, ts_headline('english', body, websearch_to_tsquery('english', 'wraparound'),
                       'StartSel=[, StopSel=], MaxWords=12, MinWords=5')
FROM kb WHERE search @@ websearch_to_tsquery('english', 'wraparound');
```

```text
 id |                  ts_headline
----+-----------------------------------------------
  2 | [wraparound]. Monitor the age of datfrozenxid
```

`ts_headline` re-parses the original text, so it is relatively expensive — apply it only to the page
of results you display. For type-ahead, prefix matching with `:*`:

```text
SELECT id, title FROM kb WHERE search @@ to_tsquery('english', 'index:*');
  3 | Choosing an index type
```

## Accents and custom configurations

```text
SELECT to_tsvector('english', 'café naïve résumé');
 'café':1 'naïv':2 'résumé':3
```

A user typing "cafe" will not find "café". The `unaccent` extension provides a dictionary you can
chain in front of the stemmer in a custom configuration:

```sql
CREATE EXTENSION unaccent;
CREATE TEXT SEARCH CONFIGURATION english_unaccent (COPY = english);
ALTER TEXT SEARCH CONFIGURATION english_unaccent
  ALTER MAPPING FOR hword, hword_part, word WITH unaccent, english_stem;

SELECT to_tsvector('english_unaccent', 'Café résumé naïve')
       @@ websearch_to_tsquery('english_unaccent', 'cafe resume');   -- t
```

Use the same configuration for the stored vector and for queries. Other configurations ship for many
languages (`\dF` lists them); for multilingual content, store a language column and build the vector
with `to_tsvector(lang::regconfig, body)`.

## Performance at scale

A 200,000-row table with a stored `search` column:

```text
-- stored vector, no index
 Parallel Seq Scan on kb_big (actual time=0.719..12.115 rows=13.33 loops=3)
   Filter: (search @@ '''wraparound'''::tsquery)
 Execution Time: 17.033 ms

-- computing the vector on the fly instead
 Parallel Seq Scan on kb_big (actual time=80.319..1053.283 rows=13.33 loops=3)
   Filter: (to_tsvector('english'::regconfig, body) @@ '''wraparound'''::tsquery)
 Execution Time: 1057.228 ms

-- stored vector with a GIN index (4656 kB)
 Bitmap Heap Scan on kb_big (actual time=0.027..0.127 rows=40.00 loops=1)
   ->  Bitmap Index Scan on kb_big_search
 Execution Time: 0.157 ms
```

Parsing and stemming text is expensive (1 second for 200,000 short documents); storing the vector
saves it, and the GIN index skips the scan entirely. An alternative to a stored column is an
expression index on `to_tsvector('english', body)`, which saves disk space but requires the query to
repeat the exact expression and recomputes vectors when rechecking or ranking.

## Where PostgreSQL search stops

Built-in full-text search does not do typo tolerance (combine it with `pg_trgm` similarity for
"did you mean"), synonyms beyond a synonym dictionary you maintain, language detection, faceting at
scale, or learning-to-rank. If search is the core of your product, a dedicated engine may be worth
its operational cost. For "search box on our app", it usually is not.

## How It Actually Works

A text search **configuration** maps each token type produced by the **parser** (words, numbers,
URLs, emails, …) to a chain of **dictionaries**. Each token is offered to the dictionaries in order;
the first that recognises it returns lexemes (or nothing, for a stop word). `english_stem` is a
Snowball stemmer that accepts everything, so it must be last in the chain — which is why `unaccent`
goes in front of it.

A `tsvector` is a sorted array of distinct lexemes, each with up to 256 positions and a weight label
(A–D) per position. A `tsquery` is a tree of operators. Evaluation checks lexeme presence, and for
phrase operators (`<->`, `<N>`) compares positions.

The GIN index stores each lexeme once with a posting list of rows containing it. A query extracts
its lexemes, fetches their posting lists, and combines them according to the boolean tree; phrase and
weight conditions are verified on the heap tuple during recheck, because positions are not stored in
the index.

## Common mistakes

- Passing user input to `to_tsquery` and getting syntax errors on input like `C++` or `"`.
- Mixing configurations: vectors built with `english`, queries with `simple`.
- Calling `to_tsvector` in the `WHERE` clause without a matching index or stored column.
- Running `ts_headline` on every match instead of the displayed page.
- Expecting stemming to produce real words or handle typos.

## Exercise

1. Load 1,000 rows of real text (for example, this course's own Markdown files, one row per lesson)
   into a table with a weighted, generated `tsvector` and a GIN index.
2. Build a search query that takes raw user input, ranks with `ts_rank_cd`, returns the top 10 with
   highlighted snippets, and never errors on odd input.
3. Create an accent-insensitive configuration and prove a search for "resume" finds "résumé".
4. Add `pg_trgm` and implement "did you mean?" suggestions for queries that return no results, using
   similarity against a table of distinct words (`ts_stat` can produce one).
