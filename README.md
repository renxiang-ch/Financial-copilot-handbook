# Supply-Chain Copilot — Handbook

A reproduction and teaching guide for the Verifiable Supply-Chain Risk Copilot.
No code lives here. Everything in this handbook targets one pinned version of
the code repository:

```
git clone https://github.com/renxiang-ch/<repo-A-name>.git
cd <repo-A-name>
git checkout v1.0-teaching
```

Every command, file path, and expected number in this handbook was checked
against that tag. If something here does not match what you see in the code,
the code is right and this handbook needs an update — please open an issue.

## Reading order

| Part | Chapter | Covers |
|---|---|---|
| I. Getting started | [01 Environment Setup](chapters/01-environment-setup.md) | Install, database, the data snapshot |
| | [02 Data Sources](chapters/02-data-sources.md) | EDGAR/XBRL/10-K, the company cluster |
| II. System | [03 Data Pipeline](chapters/03-data-pipeline.md) | Ingestion, chunking, embedding, extraction, the quality gate |
| | [04 Agent System](chapters/04-agent-system.md) | The tool loop, routing, slots, verification, clarification |
| III. Proof | [05 Evaluation](chapters/05-evaluation.md) | Eval sets, harness scoring, deterministic probes, ablations |
| IV. Optional | [06 Product Track](chapters/06-product-track.md) | The dashboard views |
| | [07 Limitations & Pitfalls](chapters/07-limitations-and-pitfalls.md) | What is known-broken, and the failure classes that recur |

Appendix: [DB Schema](appendix/a-db-schema.md) · [Expected Checkpoints](appendix/b-expected-checkpoints.md) · [Cost & Time Budget](appendix/c-cost-and-time-budget.md)

## The one thing to check before trusting a reproduction

Every published result file in the code repository carries a `db_fingerprint`.
The data snapshot (`data/seed/manifest.json`) carries the same fingerprint.
Load the snapshot, run the harness, and compare fingerprints — if they match,
your environment is correct regardless of what the numbers say next.

## Scope note

This handbook covers the product line only. The SQLLock grounding study (the
lab report / four-condition experiment) has its own repository:
`sqllock-grounding-study`. The two share a lineage but are read independently.
