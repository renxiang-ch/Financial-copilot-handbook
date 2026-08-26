# 07 · Limitations & Pitfalls

**What you'll build**: nothing — this is a reference chapter, read when a
bug looks familiar or before extending a part of the system this section
flags as weakly guarded. The closest thing to a checkpoint is
`test_constants_match_data.py` (ch.03's audit + this chapter's own
recurring-defect section share the same underlying concern), already run
as part of ch.01's test suite.

No single owning file — collected from every chapter plus the project's
internal development log; the teaching repo deliberately ships the
distilled record — its `docs/` and result files — rather than the raw log.

## A note on structure

An earlier draft of this handbook proposed a single sentence here: "every
data asset the pipeline produces has a mechanism in evaluation guarding it."
**That sentence is not true as written**, and would actively mislead a
reader into over-trusting the weaker guards below. Guard strength varies
enormously across this project — some checks are deterministic and currently
airtight, some are LLM-judged and sit inside a noise band wider than most
real changes, and at least one important behavior (multi-turn fiscal-year
inheritance, last row of the table below) is enforced by nothing stronger
than a prompt instruction the model has been observed to override.

| Data asset / behavior | What guards it | Guard strength |
|---|---|---|
| Tier-1/2 numeric answers (`eval_set.json`) | LLM-judged harness | Strong signal historically, but **saturated at 100% for months** — carries little power to catch a *new* regression, since it has had no headroom to move in either direction |
| Tool routing | `probe_router.py`, deterministic, zero LLM cost | Strong and current (20/20 at last check) — this is the guard other checks in this project should be modeled after |
| Retrieval passage-hit | LLM-judged harness | **Weak** — historical band 25–62.5%, wider than nearly every real improvement this project has measured against it (§05 §5.3) |
| Retrieval chunk-reachability (does the right chunk survive to top-k) | `probe_retrieval.py` / `probe_truncation.py`, deterministic | Strong and specific, but narrower in scope than "retrieval quality" as a whole |
| Supply-edge citation grounding | `verify_source_text_consistency()` gate at write time, plus `--audit` mode | Strong on *this specific claim* (does the stored number appear in its own cited sentence) |
| Answer-level number grounding | `grounding.py::verify_answer()` | Catches an ungrounded number; does **not** catch correct arithmetic over the wrong input in general (§04 §4.5) |
| `compute()` formula validity | `authority.py::classify()` | Detects and labels an *unsourced* formula; does not block it or correct it — detection only |
| Multi-turn fiscal-year inheritance | A prompt instruction only, no structural enforcement | **Weakest guard in this list** — measured to fail on the order of 1 in 5 runs, where the model explicitly states it is overriding the carried context and reverts to the most recent fiscal year anyway |

## The recurring defect class

**"A hardcoded inventory drifts from the data it describes"** is the single
most-repeated failure pattern in this project's history — not a one-off bug,
a *class*, recurring roughly ten times across the project's log with the
same shape each time: something enumerates a fixed list (companies, colors,
tool categories, alias mappings) at one point in time, the underlying data
grows or changes, and nothing re-checks the list against the data it was
supposed to describe. Three concrete instances, with root cause and fix,
because the pattern is easier to recognize from real examples than from the
name alone:

1. **A dashboard supplier dropdown was populated from a fixed
   ticker→color mapping table**, built for visual consistency, not as a
   source of truth for which suppliers exist. As the underlying company set
   grew, the dropdown silently continued to only offer the original
   companies — a real supplier with real data was invisible in the UI, not
   because of a bug in how it was queried, but because nothing had ever
   queried it; the dropdown's contents came from a list that was never the
   live company set to begin with. Fixed by populating the dropdown from a
   live `SELECT DISTINCT` against the company table, with the color mapping
   kept only for its actual job (assigning a color), no longer doing double
   duty as an inventory.
2. **The customer-alias dictionary in `extract_edges.py`** (ch.03 §3.4) had
   no entry for Huawei — not a US-listed ticker, so it had no natural
   canonical key — and the same real customer was written to `supply_edges`
   under three different literal strings across different extraction runs,
   including one LLM-invented ticker that happens to collide with an
   unrelated real company on a different exchange. The dictionary wasn't
   wrong when written; it was written once and never revisited as new
   entities appeared in new filings.
3. **`model_router.py`'s removed tiering table** (ch.04 §4.6) assigned every
   question category to the same cost tier — a table that, checked against
   which categories actually existed, covered 100% of them with one value,
   making the table itself dead weight rather than a real routing decision.
   Caught by asking "does this configuration make a decision, or does
   everything just fall through to the same place" before shipping it, not
   after.

The general lesson each instance shares: a fixed list is a snapshot, and a
snapshot with no mechanism to re-check itself against live data will
eventually silently disagree with it. Where this project has fixed the
pattern, the fix is consistently the same shape — replace the fixed list
with a live query, or add an explicit test that asserts the two stay in
sync (`test_constants_match_data.py` exists specifically for this).

## Known, unfixed, currently

> **This section is, itself, a live example of the drift problem above** —
> the product repo's own README "Known Limitations" section, checked while
> writing this handbook, is *already* out of date relative to the project's
> own engineering log: it still attributes a routing-cost figure to a router
> implementation that has since been rebuilt on top of `slots.py` (ch.04
> §4.3), and doesn't yet mention two items the engineering log documents as
> found and still open.

For the current state, don't read this section alone:
inside the teaching repo, the freshest ground truth is the latest regression
result files (`data/results/*_clarify_regression.json`) and the dated
postscripts in `docs/simplification-audit.md`. As of the last check, the
open items included two that share a pattern worth naming — **individually
grounded, collectively wrong**: every number involved traces to a real tool
call, so §4.5's grounding check passes cleanly, and the answer is still
wrong because the failure lives one level up, in *how* those grounded
numbers get applied or combined, not in whether they're grounded (the two
items below are different specific mechanisms within that same blind spot —
one a wrong entity/year binding, the other a wrong formula — not
duplicates of each other) — plus one that is a genuinely separate
methodological gap, not another instance of this one:

- **Relationship-direction inversion in supply-chain answers.** A supplier's
  own dependency percentage can, in a documented failure case, be applied to
  the *wrong* company's revenue figure — a ~94× magnitude error in the one
  traced instance, and every input involved was individually grounded
  (§04 §4.5's boundary: correct-looking arithmetic over the wrong binding).
  `grounding.py`'s `misbound_inputs` check now catches the specific
  sub-class where a supply-edge percentage is applied to the wrong company
  or year; it does not catch the general case of an intentionally-correct
  metric substituted for the wrong one. Residual rate on the one case this
  was tested against: roughly 10%, with a detector but not yet a structural
  fix.
- **Formula invention on derived metrics with missing inputs.** When asked
  for a metric this system has no stored definition for (e.g. book value
  per share), the model has been observed to construct a plausible-looking
  formula from available inputs rather than refusing. This is recorded and
  shown to the user (`authority.py`'s `none` bucket, §04 §4.5) rather than
  blocked — a deliberate design choice (a machine-checkable "we don't have a
  registered formula for this" label is judged more honest and useful than
  either silently trusting an unverified formula or refusing every question
  outside a fixed metric list), but it means an unsourced-and-wrong formula
  is a real, currently-possible answer shape, not a hypothetical.
- **LLM-judge same-model self-evaluation bias**, unresolved — the retrieval
  and comparison-question judge shares a model with the agent's own default.
  Not yet cross-validated against a heterogeneous judge.
