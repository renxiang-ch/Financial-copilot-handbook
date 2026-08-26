# Appendix C · Cost & Time Budget

All figures below are pulled directly from real result files' own recorded
`estimated_cost_usd`/`avg_latency_s`/`total_latency_s` fields, not estimated
— re-derive from a fresh run's own result file rather than trusting these
past the date this appendix was written (2026-08-25).

| Step | Cost | Time |
|---|---:|---:|
| Embedding generation (`embed_chunks`, one-off, 16,342 chunks) | $0 (local model, no API key) | CPU: tens of minutes on a laptop; faster with a GPU |
| Full `eval_set.json` run (30 scored items) | $0.033 | 106.2s total, 3.54s/item avg |
| `eval_set_tier3.json` run (8 items, graph-augmented) | $0.010 | 4.31s/item avg |
| `eval_set_v2.json` run (13 items) | $0.016 | 51.0s total, 3.92s/item avg |
| `eval_set_multiturn.json`, full 5-repeat pass (9 scored turns × 5 runs, why below) | $0.056 | — |

> **One-line total for "clone to first successful eval run"**: install +
> database setup + seed load + embedding generation is free (local
> model/data, no paid API calls) and dominated by wall-clock time (embedding
> generation, tens of minutes), not cost; the first `eval_set.json` run that
> actually confirms the environment is correct costs on the order of $0.03
> and takes under two minutes.

**Why this project runs the multi-turn set's `--repeat 5` deliberately
rather than by default on every change**: at roughly 5× a single-run cost
for a 9-turn set, this is small in absolute terms ($0.056) but the general
principle scales — before running any expensive, repeated-run checkpoint on
a larger set, check whether a cheaper deterministic probe (ch.05 §5.3) can
answer the same question first. Several of this project's own probes exist
specifically because they answer a question a repeated LLM-judged run would
otherwise have to spend real money to answer at all, or would answer with
less certainty for more money.
