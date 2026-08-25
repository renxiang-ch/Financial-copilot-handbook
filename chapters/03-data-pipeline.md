# 03 · Data Pipeline

Covers: `pipeline/ingest_xbrl.py`, `pipeline/ingest_text.py`,
`pipeline/ingest_item8.py`, `pipeline/embed_chunks.py`,
`pipeline/extract_edges.py`, `retrieval/{bm25,dense,hybrid}.py`.

## 3.1 XBRL ingestion → financial_facts

<!-- TODO: ingest_xbrl.py. The 24 metric labels currently served — pull with
     `SELECT DISTINCT label FROM financial_facts WHERE form='10-K'`, do not
     hardcode a list here, it will drift (this is the exact defect class
     documented in ch.07) -->

## 3.2 10-K text ingestion → filings + text_chunks

<!-- TODO: ingest_text.py, chunking strategy, the ix:nonfraction fix and why
     it mattered (see CLAUDE.md 2026-08-21 entry) -->

## 3.3 Embeddings + retrieval

<!-- TODO: embed_chunks.py (bge-small-en-v1.5, local, 384-dim), then
     retrieval/bm25.py + dense.py + hybrid.py (RRF fusion, k=60). Table
     filtering (chunk_type) and year-scoping (R2) both belong here — they are
     retrieval-time WHERE clauses, not separate systems. -->

## 3.4 Supply-chain edge extraction

<!-- TODO: extract_edges.py -- this is the pipeline's methodological center.
     Regex pre-filter -> schema-guided LLM extraction -> Pydantic validation ->
     the quality gate (verify_source_text_consistency, --audit mode). Walk
     through at least one real bug as a worked example (JBL citation bug,
     QRVO fabricated percentage, or the Huawei entity-fragmentation case --
     all documented in CLAUDE.md with full root-cause writeups, don't
     reconstruct from memory, copy the verified numbers). -->

## 3.5 Why the shipped data is a snapshot, not a re-run target

<!-- TODO: supply_edges carries hand-applied corrections; the ix:nonfraction
     fix changed candidate blocks for 32 of 54 filings after the last
     extraction run. Re-running extraction will NOT reproduce the shipped
     table. Say this plainly -- it is a real teaching point about the gap
     between "reproducible" and "re-runnable" in data pipelines. -->
