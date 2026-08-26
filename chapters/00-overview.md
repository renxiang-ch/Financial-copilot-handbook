# 00 · Overview

**Verifiable Supply-Chain Risk Copilot** — an analyst-facing agent that answers
questions over SEC 10-K filings with numbers that trace back to SQL/XBRL and
relationships that trace back to a specific disclosure sentence, for the
Apple supply-chain cluster (AAPL plus 14 suppliers/peers spanning both named
customer-concentration disclosures and diversified companies with none).

```
User → Streamlit (chat + dashboard) → FastAPI → Agent loop
                                                    │
                        ┌───────────────┬───────────┴──────────┬──────────────┐
                        │               │                      │              │
                 query_financials  retrieve_text           graph_query    compute
                        │               │                      │              │
                        └───────────────┴──────────┬───────────┴──────────────┘
                                                     ▼
                                    PostgreSQL + pgvector
                          (financial_facts · text_chunks · supply_edges)
                                                     ▲
                                    EDGAR ingestion pipeline
                       (ingest_financial_facts · ingest_text · extract_edges)
```

Every number the agent states comes from `query_financials`/`compute`
(SQL, never the model's own arithmetic — the project's one non-negotiable
rule). Every relationship claim comes from `graph_query` against
`supply_edges`, each row carrying the verbatim 10-K sentence it was
extracted from. A verification layer (`agent/grounding.py`,
`agent/authority.py`) checks after the fact that every number in an answer
actually traces back to a tool call, and flags what it cannot source rather
than silently trusting the model.

## Two lines of work, two repositories

- **Product** (this handbook covers it): `Financial-Report-Research-Copilot`
  — the agent, the eval harness, the dashboard. This handbook targets the
  `v1.0-teaching` tag: a curated rebuild of the development repo (cut
  2026-08-26), pruned of development-history residue; see the README.
- **SQLLock study**: `sqllock-grounding-study` — a separate, later controlled
  experiment asking a narrower question ("across four progressively more
  structured ways of giving an LLM agent the same SEC filing facts, where
  does the accuracy gain actually come from") on top of a pinned snapshot of
  this same codebase. Different repo, different README, different report —
  read it on its own, not as a continuation of this handbook.

## The core numbers, checked live against the database on 2026-08-25

| | |
|---|---|
| Companies | 15 (AAPL hub + 14 suppliers/peers) |
| Filings | 148 |
| XBRL financial facts | 38,966, across 24 canonical metric labels |
| Text chunks | 16,342 (11,990 prose / 4,352 table), all embedded |
| Supply-chain edges | 128 total, 103 named |
| Frozen eval set (`eval_set.json`, v1.3, 30 scored + 3 retired) | Tier-1/Tier-2/input-fetch/refusal all 100%; retrieval passage hit 42.9% (noise band, see ch.05); overall 86.7% |
| Tier-3 graph ablation (`eval_set_tier3.json`, 8 items) | graph-augmented 100% vs. `--no-graph` baseline 12.5% — **+87.5pp**, the graph layer's measured contribution |
| Unit tests | 128 |

(Was 129 before `v1.0-teaching`: two ingestion-drift guard tests merged into
one when the two ingestion scripts they each pinned became one script, ch.03.)

> **These numbers will have moved by the time you read this.** Re-check with
> `copilot.eval.harness --out <path>` against this repo's live database
> rather than trusting the table above past the date it was checked.

## A note on how this handbook was built

Every fact, file path, and number here was checked against the running
repository (a live `psql` query against the actual database, or a direct
read of the current source file) rather than reconstructed from the
project's internal development log from memory alone — that log is
detailed and reliable as a *history*, but this handbook's job is to describe
the *current* state, and the two are not always the same thing on a project
that has been under active, daily development. Where a check could not be
completed, the chapter says so explicitly rather than presenting a guess as
verified.
