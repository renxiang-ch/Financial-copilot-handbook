# 03 · Data Pipeline

Covers: `pipeline/ingest_financial_facts.py`, `pipeline/ingest_text.py`,
`pipeline/ingest_item8.py`, `pipeline/embed_chunks.py`,
`pipeline/extract_edges.py`, `retrieval/{bm25,dense,hybrid}.py`.

## 3.1 XBRL ingestion → `financial_facts`

`ingest_financial_facts.py` pulls each company's `companyfacts` payload from EDGAR and
filters it before anything reaches the database — the project's
"numbers never come from LLM arithmetic" rule starts here, one layer before
the agent even exists. A fixed whitelist (`us-gaap` namespace only,
`10-K`/`10-Q` form only) narrows hundreds of raw tags down to the current 24
canonical labels served (`SELECT DISTINCT label FROM financial_facts WHERE
form='10-K'` — check this directly rather than trusting a hardcoded list
anywhere, including this sentence, past the date it was written).

This whitelist is also the mechanism behind a real, measured coverage gap: a
number can be fully XBRL-tagged in the source filing and still never reach
`financial_facts`, simply because its specific tag isn't in the dict. This
is not a bug — it's the tradeoff of a curated schema over an open-ended one
— but it means "not in the database" and "not disclosed" are different
claims, and the agent's refusal language is written to not conflate them.

Two real defects were found and fixed in this layer (2026-07-30): a
**duration mismatch** (EDGAR sometimes reports a quarterly-duration fact
under the annual figure's tag, with a matching period_end — caught by
requiring 10-K duration facts to span 355–380 days) and a **tag collision**
(multiple XBRL tags mapping to one label, where for several labels the
collision was two genuinely different financial concepts, not synonyms —
resolved by splitting four labels and giving three an explicit tag-priority
order). The fixes originally lived in a separate `fix_financial_facts.py`
alongside the original, uncorrected `ingest_xbrl.py` — which meant every
re-run was one wrong module name away from silently re-inserting the rows
the corrected logic rejects. As of `v1.0-teaching` the two are merged into
the single `ingest_financial_facts.py`: corrected logic, `companies`
upsert, and empty-database bootstrap in one script, with no other ingestion
path left to reach for. The module's docstring carries the full defect
history; a drift guard in `tests/test_constants_match_data.py` pins its
cluster dict to the canonical source so a local copy can never quietly
reappear.

## 3.2 10-K text ingestion → `filings` + `text_chunks`

`ingest_text.py` downloads each filing's HTML, sections it (Business / Risk
Factors / MD&A / Item 8 financial statements), and chunks it for retrieval.

**A real, fixed extraction defect worth understanding, not just noting**:
EDGAR's HTML filings mark inline numeric values with `<ix:nonfraction>` tags
for XBRL tagging purposes; an earlier version of this pipeline's text
extraction did not account for this markup, which silently affected which
text blocks were captured as extraction candidates downstream. The fix
changed the candidate text blocks for **32 of the 54 filings** behind the
supply-chain graph's named edges — a large blast radius that was measured,
not assumed, by re-running candidate extraction against all 54 filings both
before and after the fix and diffing the results. This is exactly the kind
of defect ch.07 discusses as a class: a narrow-looking parsing assumption
that turns out to have wide, silent reach once actually measured.

## 3.3 Embeddings + retrieval

`pipeline/embed_chunks.py` generates local embeddings
(`BAAI/bge-small-en-v1.5`, 384 dimensions, no API key) for every chunk. Three
retrieval modules compose on top:

- `retrieval/bm25.py::retrieve_text()` — lexical (term/number overlap).
- `retrieval/dense.py::retrieve_dense()` — semantic (embedding cosine
  similarity), catching paraphrases BM25 misses.
- `retrieval/hybrid.py::retrieve_hybrid()` — merges the two rankings via
  **Reciprocal Rank Fusion** (`k=60`, the standard smoothing constant from
  Cormack, Clarke & Buettcher, SIGIR 2009), combining by rank position
  rather than raw score so the two very differently-scaled rankings don't
  need score normalization.

Two retrieval-time filters are `WHERE` clauses on this same pipeline, not
separate systems: `include_tables` (excludes `chunk_type='table'` rows by
default — measured to carry almost none of the golden citations the eval
set checks for, while contributing disproportionately to over-length chunks
that get silently truncated by the embedding model's 512-token window) and
`fiscal_year` (scopes retrieval to a specific filing year when the question
names one, falling back to unscoped search for trend-style questions that
need to see every year). Both are documented, deliberate narrowings of what
`retrieve_text` searches — see ch.07 for the measured tradeoffs each one
costs.

## 3.4 Supply-chain edge extraction

`extract_edges.py` is the pipeline's methodological center. Pipeline:
**regex pre-filter** (`_extract_item8_candidates()` narrows a filing's Item 8
financial-notes section to text blocks containing "%" + "revenue" + a
company name) → **schema-guided LLM extraction** (`_extract_from_chunk()`,
constrained by a Pydantic schema, `EdgeCandidate`) → **validation** →
**a quality gate that can reject a write**, not just log a warning:
`verify_source_text_consistency()` checks that the extracted `revenue_pct`
(or, for a threshold-only disclosure, the threshold wording itself) actually
appears in the quoted `source_text` before the row is allowed into
`supply_edges`. A candidate that fails this check does not get silently
dropped either — `audit_existing_edges(conn, include_unnamed=False)` (the
`--audit` CLI mode) can be run at any time to scan the whole table and
report which rows are clean versus suspicious, independent of whether the
gate was in place when a given row was written.

**A worked example, because the mechanism is easier to understand from a
real failure than from the code alone.** An audit of the existing
`supply_edges` table (2026-07-15/16)
found three distinct, real defect classes in already-written rows, none of
them hypothetical:

1. **Citation pointed at the wrong sentence, number was actually correct**
   (JBL→AAPL, several fiscal years). The extraction had quoted an aggregate
   "our five largest customers" sentence instead of the row of a nearby
   table that actually stated Apple's specific percentage — the number in
   the database was right, but the `source_text` couldn't prove it.
2. **The number itself was wrong, not just the citation** (QRVO→AAPL FY2018/
   FY2019: stored as 12%/11%, actual disclosed values were 36%/32%). Found
   by noticing the full multi-year trend line was smooth except for a two-
   year dip, then checking the original filing directly — this is the class
   `verify_source_text_consistency()` exists specifically to catch, since a
   number that isn't grounded in its own cited sentence at write time now
   fails the gate rather than requiring an analyst to notice a suspicious
   trend line after the fact.
3. **Entity fragmentation**: the same real-world customer (Huawei — not
   itself a public-company ticker) got written to `supply_edges` under three
   different literal strings across different extraction runs
   (`"Huawei"`, `"Huawei Technology Co., Ltd."`, `"002502.SZ"` — the last one
   an LLM-invented ticker, since Huawei has none), because the customer-alias
   dictionary had no entry for it. Fixed by adding the alias mapping and
   manually merging the fragmented rows — the general lesson (a hardcoded
   alias/lookup table drifts from what the data actually contains) recurs
   across this project and is the subject of ch.07's dedicated section.

## 3.5 Why the shipped data is a snapshot, not a re-run target

`supply_edges` carries the hand-applied corrections from the worked example
above; the `ix:nonfraction` fix (§3.2) changed the extraction candidate set
for 32 of 54 filings *after* the last full extraction run. Re-running
`extract_edges.py` today would **not** reproduce the shipped table — it
would produce a different one, of unknown-but-probably-mixed quality,
since the newly-visible text from the `ix:nonfraction` fix hasn't itself
been re-extracted and re-audited yet. This is a real, deliberate distinction
this project draws between two words that sound like synonyms:

- **Reproducible** — a fixed artifact plus a fingerprint that lets you
  confirm you're looking at the same one. The shipped `data/seed/` snapshot
  is this.
- **Re-runnable** — running the pipeline again from scratch produces the
  same result. The pipeline currently is *not* this, for the reasons above.

Conflating the two is a common failure mode in data-engineering
reproducibility claims generally, and this project says plainly which one it
is rather than letting a reader assume the stronger claim.
