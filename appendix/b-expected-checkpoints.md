# Appendix B · Expected Checkpoints

The judgment call every checkpoint here follows: report a number as a fixed
target only where the harness is actually deterministic at that number
(Tier-1, Tier-2, refusal accuracy have been saturated at 100% for months).
Where a metric has a documented noise band, report the band, not a point
value — a point value invites a reader to treat a within-noise fluctuation
as a broken reproduction. All numbers below were read from the live database
and from `data/results/eval_results_p3_regression.json` /
`eval_results_t3_p3_regression.json` (the most recent full regression pair
on record at the time of writing) on 2026-08-25 — re-check against fresh
result files before trusting these past that date.

## Database fingerprint (after `seed --load`)

From `data/seed/manifest.json`:

```json
{
  "companies": 15,
  "filings": 148,
  "financial_facts": 38966,
  "text_chunks": 16342,
  "supply_edges": 128,
  "named_edges": 103,
  "embedded_chunks": 16342
}
```

`embedded_chunks` will read `0` immediately after `seed --load` and only
match `text_chunks` after `embed_chunks` (ch.01 §1.4) completes — a mismatch
here before that step is expected, not an error.

## Frozen eval set (`eval_set.json`, v1.3, 30 scored + 3 retired)

| Metric | Expected |
|---|---|
| Tier-1 accuracy | 100% |
| Tier-2 accuracy | 100% |
| Tier-2 input fetch | 100% |
| Refusal accuracy | 100% |
| Retrieval passage hit | 25–62.5% (band, not a point — last measured 42.9%) |
| Overall | 86.7% at last measurement |

> **Retrieval passage hit is a band, not a point value** — it has run
> 25–62.5% across otherwise-identical runs historically. A number inside
> that band is not evidence of anything having broken; see ch.05 §5.1/§5.3
> before investigating a "regression" here.
>
> **"Overall" moves with retrieval's noise**, since the numeric and refusal
> metrics above it are saturated at 100%. Don't read a change in "Overall"
> as informative on its own — check which sub-metric actually moved first.

## Tier-3 (`eval_set_tier3.json`, 8 items)

| Arm | Overall accuracy |
|---|---|
| Graph-augmented (default) | 100% (8/8) |
| `--no-graph` baseline | 12.5% (1/8) |
| **Delta — the graph layer's measured contribution** | **+87.5pp** |

## Router probe (`probe_router.py`)

Deterministic, zero LLM cost — should be exact. 20/20 at last check. If this
is not 20/20 on a fresh run, treat it as a real regression to investigate
immediately, not as noise; nothing about this probe's design permits
run-to-run variance.

## Multi-turn (`eval_set_multiturn.json`, 5 conversations / 9 scored turns)

**A single run is not a valid checkpoint for this set** — the effect
context-inheritance (ch.04 §4.4) is meant to produce did not separate from
noise at n=1 in this project's own history (ch.05 §5.5). Use
`--repeat 5` and compare the *distribution* across the 5 runs, not one run's
score:

| | Turn accuracy (mean, range across 5 runs) | Year binding |
|---|---|---|
| Inheritance disabled (`--no-inherit`) | 82.2% (77.8–88.9%) | 82.5% (75.0–87.5%) |
| Inheritance enabled (default) | 97.8% (88.9–100%) | 97.5% (87.5–100%) |

The ranges overlap at their extremes at n=5 — this is stated plainly rather
than rounded away, because it is the honest state of the evidence; the
effect is real (it isolates to the two turns specifically designed to expose
context loss, both of which move consistently across all 5 runs) but is not
"clean separation with no overlap," and a checkpoint run that lands at the
edge of either range is not automatically a failure.
