# Appendix A · Database Schema

This appendix does not duplicate the schema as a static copy — a copied-out
SQL file drifts from the real one the moment either changes (ch.07's
recurring-defect class, applied to documentation itself). Instead:

**Authoritative source**: `src/copilot/storage/schema.py` in repo A. Read
`CREATE_TABLES_SQL` and the `migrate_*` functions `create_tables()` calls —
several columns below were added by a migration after the table's original
creation, and the migration functions are the actual record of when and why.

**Schema actually in effect**, captured live via `psql -d financial_copilot
-c '\d <table>'` on 2026-08-25 — treat this as a point-in-time reference;
`schema.py` is the source of truth if the two ever disagree:

```
Table "public.companies"
   Column   |           Type           | Nullable | Default
------------+--------------------------+----------+---------
 ticker     | text                     | not null |
 name       | text                     | not null |
 cik        | text                     | not null |
 created_at | timestamp with time zone |          | now()
PRIMARY KEY (ticker); UNIQUE (cik)

Table "public.filings"
   Column    |  Type   | Nullable | Default
-------------+---------+----------+---------
 accn        | text    | not null |
 ticker      | text    | not null |   -- FK -> companies(ticker)
 form        | text    | not null |
 filed_date  | date    | not null |
 fiscal_year | integer |          |
 doc_url     | text    | not null |
PRIMARY KEY (accn)

Table "public.financial_facts"
    Column     |  Type   | Nullable | Default
---------------+---------+----------+---------------------------
 id            | bigint  | not null | (sequence)
 ticker        | text    | not null |   -- FK -> companies(ticker)
 tag           | text    | not null |   -- raw XBRL tag
 label         | text    | not null |   -- canonical label (24 distinct values, ch.02)
 value         | numeric | not null |
 unit          | text    | not null |
 period_end    | date    | not null |
 fiscal_year   | integer |          |
 fiscal_period | text    |          |
 form          | text    |          |
 accn          | text    | not null |
 period_start  | date    |          |   -- added by a migration (validates duration facts, ch.03 §3.1)
UNIQUE (ticker, tag, period_end, accn); indexed on (ticker, tag) and (ticker, fiscal_year)

Table "public.text_chunks"
    Column    |    Type     | Nullable | Default
--------------+-------------+----------+---------------
 id           | bigint      | not null | (sequence)
 accn         | text        | not null |   -- FK -> filings(accn)
 ticker       | text        | not null |
 section      | text        |          |
 chunk_index  | integer     | not null |
 text         | text        | not null |
 token_count  | integer     |          |
 embedding    | vector(384) |          |
 chunk_type   | text        |          | 'text'   -- 'text' | 'table', added by migration (ch.03 §3.3)
 embedding_v1 | vector(384) |          |   -- backup column, see note below
UNIQUE (accn, chunk_index); HNSW index on embedding (cosine); indexed on accn, ticker
```

`embedding_v1` was added live by a later re-embedding fix and is **not** in
`schema.py`'s base `CREATE_TABLES_SQL` — check `schema.py`'s `migrate_*`
functions for its actual provenance before assuming it is load-bearing for
anything downstream.

```
Table "public.supply_edges"
      Column       |           Type           | Nullable | Default
-------------------+--------------------------+----------+------------
 id                | integer                  | not null | (sequence)
 supplier_ticker   | text                     | not null |
 customer_ticker   | text                     | not null |
 revenue_pct       | double precision         |          |
 fiscal_year       | integer                  |          |
 disclosure_status | text                     |          | 'named'   -- 'named' | 'inferred' | 'unnamed'
 accn              | text                     |          |
 chunk_id          | integer                  |          |   -- no FK constraint to text_chunks (see caveat below)
 extracted_at      | timestamp with time zone |          | now()
 source_text       | text                     |          |   -- verbatim disclosure sentence(s), ch.03 §3.4
 threshold_only    | boolean                  |          | false     -- ch.03 §3.4, ch.06's floor-estimate section
UNIQUE (supplier_ticker, customer_ticker, fiscal_year); indexed on supplier_ticker, customer_ticker, fiscal_year
```

**One schema gap worth knowing before writing code against `supply_edges`,
checked live rather than left as a hypothetical**: `chunk_id` has no
foreign-key constraint to `text_chunks.id` — it is an informal pointer, not
an enforced one. Checked directly against the live database (2026-08-25):

```sql
SELECT count(*) FROM supply_edges e LEFT JOIN text_chunks c ON e.chunk_id = c.id
WHERE e.chunk_id IS NOT NULL AND c.id IS NULL;
-- 33
SELECT count(*) FROM supply_edges WHERE chunk_id IS NOT NULL;
-- 33
```

> **Every single non-null `chunk_id` in the current database points at a
> row that no longer exists** — not a rare edge case, the entire non-null
> population. This is a real, silent consequence of the schema gap
> (`text_chunks` rows can be deleted or renumbered by a re-indexing/
> re-embedding pass, as has happened during this project's retrieval-layer
> work, with nothing enforcing that `supply_edges.chunk_id` stays valid) —
> and it is currently harmless only because nothing in the running product
> reads `supply_edges.chunk_id` for anything; every citation the product
> actually displays comes from `accn`/`source_text` on the same row, not
> from a `text_chunks` join. **Do not add a feature that assumes `chunk_id`
> resolves to a live row without checking first** — it does not, for any
> row in the current database.
