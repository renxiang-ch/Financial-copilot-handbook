# Financial-Copilot — Handbook

**What this teaches**: how to build a tool-calling LLM agent that answers
financial-analyst questions over SEC 10-K filings with every number and
every relationship traceable to a specific document — structured data
access (SQL/XBRL) alongside hybrid retrieval alongside a hand-extracted
relationship graph, one router deciding which tool a question actually
needs, and an evaluation harness that catches regressions a single test
run can't. It is a working, opinionated answer to "how do you make a RAG
agent trustworthy enough to cite," built and hardened against real,
documented failures rather than designed from a clean-room spec.

**What you'll build**, following chapters 01–05 in order: a running agent
(chat + dashboard) answering questions like *"How dependent is Cirrus Logic
on Apple, and how has that changed since 2022?"* with a cited, SQL-verified
number — plus the eval suite that would have caught it if the agent got
that wrong. Chapter 00 walks one such question through every layer of the
system before you touch a command, so you have a mental model before you
have a terminal open.

**Start here** → [00 · Overview](chapters/00-overview.md), then follow the
reading order below. Every chapter states what you'll be able to build or
verify by its end, and ends with a checkpoint you can run to confirm you
got there.

Everything in this handbook targets one pinned version of the code
repository — no code lives in this repo itself:

```
git clone https://github.com/renxiang-ch/Financial-Report-Research-Copilot.git
cd Financial-Report-Research-Copilot
git checkout v1.0-teaching
```

## Reading order

| Part | Chapter | Covers |
|---|---|---|
| I. Getting started | [01 Environment Setup](chapters/01-environment-setup.md) | Install, database, the data snapshot |
| | [02 Data Sources](chapters/02-data-sources.md) | EDGAR/XBRL/10-K, the company cluster |
| II. System | [03 Data Pipeline](chapters/03-data-pipeline.md) | Ingestion, chunking, embedding, extraction, the quality gate |
| | [04 Agent System](chapters/04-agent-system.md) | The tool loop, routing, slots, verification, clarification |
| III. Proof | [05 Evaluation](chapters/05-evaluation.md) | Eval sets, harness scoring, deterministic probes, ablations |
| IV. Optional | [06 Product Track](chapters/06-product-track.md) | The dashboard views |
| | [07 Reliability Boundaries & Failure Modes](chapters/07-limitations-and-pitfalls.md) | Where "verifiable" stops meaning "always correct," precisely |

Appendix: [DB Schema](appendix/a-db-schema.md) · [Expected Checkpoints](appendix/b-expected-checkpoints.md) · [Cost & Time Budget](appendix/c-cost-and-time-budget.md)

## Where `v1.0-teaching` comes from

Cut 2026-08-26 on a rebuilt, curated repository: the development repo (169
tracked files, three months of intermediate result files and one-time
scripts) was pruned to the 106 files a reader actually needs — full source,
tests, the seed snapshot, all six eval sets, and a curated evidence chain of
25 result files. Every command, file path, and number in this handbook was
checked directly against the running repository on 2026-08-25 and re-synced
against the curated repo on 2026-08-26. One code change landed with the
curation and is reflected in ch.03: the two XBRL ingestion scripts were
merged into `ingest_financial_facts.py`. The project's raw day-by-day
development log is deliberately not part of the teaching repo — this
handbook and the repo's own `docs/` (the case study, the audit) carry the
distilled record, which is the version a reader actually needs.

If something here does not match what you see in the code, the code is
right and this handbook needs an update — please open an issue.

## The one thing to check before trusting a reproduction

Every published result file in the code repository carries a `db_fingerprint`.
The data snapshot (`data/seed/manifest.json`) carries the same fingerprint.
Load the snapshot, run the harness, and compare fingerprints — if they match,
your environment is correct regardless of what the numbers say next.

## Scope note

This handbook covers the product line only. The SQLLock grounding study (the
lab report / four-condition experiment) has its own repository:
`sqllock-grounding-study`. The two share a lineage but are read independently.
