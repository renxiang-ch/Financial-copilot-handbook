# Appendix B · Expected Checkpoints

The judgment call every checkpoint here follows: report a number as a fixed
target only where the harness is actually deterministic at that number (Tier-1,
Tier-2, refusal accuracy have been saturated at 100% for months). Where a
metric has a documented noise band, report the band, not a point value — a
point value invites a reader to treat a within-noise fluctuation as a broken
reproduction.

## Database fingerprint (after `seed --load`)

```
<TODO: paste data/seed/manifest.json's "fingerprint" object at tag time>
```

## Frozen eval set (`eval_set.json`)

| Metric | Expected |
|---|---|
| Tier-1 accuracy | <TODO> |
| Tier-2 accuracy | <TODO> |
| Tier-2 input fetch | <TODO> |
| Refusal accuracy | <TODO> |
| Retrieval passage hit | <TODO> — historically 25–62.5%, report as a band |
| Overall | <TODO> |

## Tier-3 (`eval_set_tier3.json`)

<!-- TODO: graph-augmented vs --no-graph, both numbers, the delta -->

## Router probe (`probe_router.py`)

<!-- TODO: deterministic, should be exact — 20/20 at last check -->

## Multi-turn (`eval_set_multiturn.json`)

<!-- TODO: this one specifically needs --repeat N with N>1 in the checkpoint
     instructions -- a single run is not a valid checkpoint for this set, see
     ch.05 §5.5 -->
