# 07 · Reliability Boundaries and Failure Modes

**What you'll build**: nothing new — this chapter no longer introduces
system capabilities. It answers a harder, more important question
directly:

> **When we call this agent "verifiable," where does that guarantee
> actually stop?**

The system reduces the model's opportunity to state unverified facts
through SQL/XBRL access, a structured relationship table, tool-call
traces, and post-hoc verification. But "a number traces back to a real
tool call" is not the same claim as "the whole answer is semantically
correct." This chapter does not lump every gap into one bucket labeled
"bug." It separates four different kinds of boundary:

- issues already **structurally prevented**
- issues that are **detected, but not yet fully blocked**
- **residual risk the system accepts and surfaces to the user**, by design
- **open problems** in evaluation or data coverage, not yet resolved

The closest thing to a runnable checkpoint for this chapter is
`test_constants_match_data.py` (§7.9), already part of ch.01's test suite —
no new command here.

## 7.1 What the system actually guarantees

The current system provides several strong guarantees. Every financial
number must come from `query_financials` or `compute`, never generated
from the model's memory. Every supply-chain relationship must come from
`graph_query` against `supply_edges`, and each edge carries the source
SEC filing's accession number and `source_text`.

Second, `grounding.py::verify_answer()` checks, after the answer is
generated, whether every number and citation in the final text actually
appears in this run's own tool-call trace. If the model states a number no
tool result ever produced, this layer catches it.

The write path for supply-chain relationships carries an additional data
quality gate: `verify_source_text_consistency()` requires that an
extracted percentage or threshold phrase actually be findable in that
edge's own `source_text` before the row is allowed into the database.

Together, these mechanisms answer one question fairly strongly:

> **"Does this number or relationship have a traceable source?"**

They do not fully answer a harder one:

> **"Did the model combine these real pieces of evidence with the correct
> semantic relationship between them?"**

That gap is this system's most important current reliability boundary,
stated once here as a summary before the sections below unpack it:

```text
It guarantees:
  - numbers trace to tools
  - relationship edges trace to filings
  - certain known binding errors are detected

It does NOT guarantee:
  - every grounded number is semantically correct
  - every valid computation uses the right formula
  - retrieval always surfaces the best evidence
```

## 7.2 Grounded ≠ Correct

The most important remaining risk class can be named precisely:

> **Individually grounded, collectively wrong** — every fact involved is
> real, and the combination can still be wrong.

A documented, traced example: a supplier's dependency percentage on Apple
was real. A company's revenue figure was real. The multiplication
`compute()` performed was arithmetically correct. But the model bound the
*supplier's own* dependency percentage to a *different company's* revenue
figure — a roughly 94× magnitude error in the resulting number.

Every individual number in that trace came from a real tool call, so a
conventional "is this number grounded" check still passes cleanly.

`grounding.py` now has a targeted detector for this specific failure —
`misbound_inputs` — which checks whether a supply-edge percentage has been
applied to the wrong company or the wrong fiscal year. This closes the one
sub-case that was actually traced and measured.

But this remains, honestly:

**Detected, not yet fully, structurally blocked.**

`misbound_inputs` covers the known company/year-binding sub-case; it does
not prove every possible semantic-binding error is caught. If the model
retrieves a real but conceptually wrong financial metric, a general
grounding check may still consider it "sourced."

So the boundary, stated as plainly as it can be:

> **Evidence grounding is necessary for correctness. It is not sufficient
> for it.**

## 7.3 Formula invention on derived metrics

A related but mechanically different problem shows up in derived metrics.

When a user asks for a metric this system has no registered definition
for — book value per share, for example — the model does not always
refuse. It sometimes constructs a plausible-looking formula from whatever
legitimate financial data is available.

This is not arithmetic hallucination. The expression `compute()` evaluates
can be entirely correct, and every input number can genuinely come from
the database. The real question is different:

> **Is the formula itself sourced?**

So `authority.py` does not attempt to enumerate and validate every possible
financial formula — an unbounded set. It answers a narrower, machine-
checkable question instead: **where did this formula come from?**
Three buckets:

- `question` — the question itself specified the arithmetic
- `registry` — the formula matches one this project's own trace history
  has repeatedly confirmed correct
- `none` — the system cannot confirm where the formula came from

A `none` classification is not automatically treated as an error and does
not block the answer. It is surfaced to the reader as an **unsourced
derivation** instead. This is a deliberate design choice: the system opts
to detect and plainly disclose "this formula has no registered basis"
rather than pretend it can verify the semantic correctness of arbitrary
financial formulas.

So this item belongs to a distinct category:

**Detected, and the residual risk is deliberately accepted.**

An unverified, and possibly actually wrong, formula can still reach the
final answer — the system just no longer presents it as something that
has been checked.

## 7.4 Multi-turn context carrying is not yet a hard constraint

Multi-turn context inheritance does not rely entirely on the prompt.
`conversation.py` extracts slots (company, fiscal year, etc.) from prior
turns and injects them into the current turn as a structured context
block; when a pronoun is involved, it substitutes the carried company in
and re-parses the sentence from scratch. **Carrying context is
structural**, not a hope expressed in prose.

The weak point is downstream of that:

> Whether the model actually *honors* the fiscal year it was handed is
> still not enforced at the tool-call layer.

In this project's own multi-turn evaluation, the model has explicitly
overridden the carried fiscal year and re-selected the most recent year in
the database instead — observed on the order of once every five runs.

So the current state is:

**Context carrying is structured. Context compliance still partly depends
on the model following a prompt instruction.**

A stronger future fix would bind the inherited fiscal year directly into
the tool call's parameters, rather than continuing to leave the model free
to override it.

## 7.5 Retrieval's evaluation signal is still weak

The retrieval passage-hit metric has ranged **25%–62.5%** across otherwise-
identical historical runs — a band wider than most of the real engineering
improvements this project has ever measured. So:

> A single run's retrieval score moving up or down cannot, by itself, be
> read as the system having genuinely improved or regressed.

The project isolates specific mechanisms deterministically instead, via
`probe_retrieval.py`, `probe_truncation.py`, and `probe_year_scope.py` —
checking, for example, whether the right chunk still reaches the top-k
window, whether a chunk gets silently truncated by the embedding model's
token limit, and whether the fiscal-year filter actually narrows results
to the correct filing.

These probes are strongly deterministic about the specific mechanism each
one names. But it has to be said plainly:

> They prove the mechanism each probe covers is working. They do not prove
> the overall quality of retrieval as a whole.

Retrieval therefore remains the comparatively weakest-measured part of the
current evaluation suite.

## 7.6 Same-model LLM-judge bias is still unresolved

Some qualitative retrieval and comparison-question scoring still uses an
LLM judge, and that judge shares a model family with the agent's own
default. This carries an unquantified risk:

> The judge may systematically favor answers that resemble its own
> generation style.

No heterogeneous-judge cross-validation has been run yet, so the actual
size of this bias is unknown. This remains an **open evaluation-
methodology limitation**. Until it is resolved, LLM-judged metrics should
be treated as supporting evidence — not given the same weight as
deterministic numeric or probe results.

## 7.7 Data-coverage boundary: absent from the database ≠ undisclosed by the SEC

`financial_facts` is not a full mirror of a company's SEC XBRL data. The
ingestion pipeline only maps a chosen set of `us-gaap` tags to this
system's current canonical metric labels — a choice that keeps downstream
querying and entity normalization simple, at the necessary cost of
coverage. So:

> `query_financials()` returning "not found" only means "this metric isn't
> in the current structured schema." It does not mean "the company never
> disclosed this."

The agent's refusal wording is written to keep these two claims separate,
but the underlying boundary remains real and worth restating explicitly.

The same logic applies to the supply-chain relationship data, which is
built almost entirely from supplier-side customer-concentration
disclosures. A question like:

> "What percentage of Apple's procurement comes from Qorvo?"

is unanswerable from current data, even though the system knows what
percentage of *Qorvo's* revenue comes from Apple. This is not a retrieval
failure. It is that **the evidence, by construction, only faces one
direction.**

## 7.8 A reproducible snapshot is not the same as a re-runnable extraction pipeline

This is one of the most important reproducibility boundaries in the
current data pipeline.

The teaching repo ships `data/seed/` — a fixed database snapshot tied to
every published result via a `db_fingerprint`. So:

> **The current project is reproducible.** A reader can load the same
> snapshot and confirm they're working from the same data state every
> published result was measured against.

The extraction pipeline, however, is **not**, in the strict sense,
**re-runnable from scratch**. `supply_edges` carries corrections applied
by hand after manual review; the `ix:nonfraction` extraction fix landed
*after* the last full edge-extraction run and changed the candidate text
blocks for 32 of the 54 filings behind those edges; the newly visible
candidate text has not itself been re-extracted and re-audited.

So running `extract_edges.py` again today is **not guaranteed to
reproduce the shipped `supply_edges` table.**

This is exactly why this handbook draws the distinction plainly rather
than letting a reader assume the stronger claim:

**Reproducible** — a fixed artifact plus a fingerprint that lets you
confirm you're looking at the same data state a result was measured
against.

**Re-runnable** — running the pipeline again from raw source data
produces, deterministically, the same final artifact.

The current system satisfies the first. It does not yet satisfy the
second.

## 7.9 A recurring engineering failure pattern: hardcoded inventory drift

Beyond specific feature limitations, this project's development history
has repeatedly hit one more general engineering problem:

> **A hardcoded list is correct when it's written. When the underlying
> real data changes, nothing forces the list to keep up.**

This pattern has shown up in more than one, seemingly unrelated module.

**The customer-alias dictionary.** Huawei was initially missing from
`CUSTOMER_ALIASES`. Different extraction runs consequently wrote the same
real-world entity to the database under three different literal strings:
`Huawei`, `Huawei Technology Co., Ltd.`, and `002502.SZ` — the last one an
LLM-invented ticker that doesn't actually belong to Huawei. Fixed by
adding the alias mapping and merging the fragmented historical rows.

**The model-router tiering table.** An earlier version of `model_router.py`
maintained a table assigning different question categories to different
model tiers. Checked against which categories actually existed, every
single one fell through to the same tier — the table existed, but was
never actually making a decision. It was deleted rather than kept for the
sake of looking architecturally complete.

Both examples share the same shape:

```text
Hardcoded snapshot
        |
        v
Underlying state changes
        |
        v
No consistency mechanism
        |
        v
Silent drift
```

The project has settled on two general responses to this pattern:

1. Where an inventory can be derived from live data, derive it from live
   data directly, rather than snapshotting it.
2. Where a hardcoded copy has to exist, add a test that asserts it still
   matches the live state.

`test_constants_match_data.py` is the concrete implementation of the
second strategy — and the closest thing this chapter has to a checkpoint
you can run.

## 7.10 Current risk status

| Issue | Type | Current status | Existing mitigation | Residual boundary |
|---|---|---|---|---|
| Company/year binding errors | Agent correctness | **Detected, not fully blocked** | `misbound_inputs` | Other semantic-binding errors can still slip through |
| Model invents an unregistered formula | Agent correctness | **Detected, residual risk accepted** | `authority:none` warning | A wrong formula can still reach the user |
| Multi-turn fiscal year gets overridden | Context | **Open problem** | Structured context carry | Not yet enforced at the tool-parameter layer |
| High retrieval-metric variance | Evaluation | **Mitigated, not resolved** | Deterministic probes | Overall retrieval quality still carries noise |
| Same-model LLM judge | Evaluation | **Open problem** | None yet | Heterogeneous-judge cross-validation not done |
| Limited XBRL canonical-schema coverage | Data coverage | **Design boundary** | Refusal wording distinguishes unavailable vs. undisclosed | Some real disclosures never reach SQL |
| Supplier-side evidence can't answer customer-side procurement questions | Evidence scope | **Design boundary** | Router refuses directly | Current data cannot fill this gap |
| Edge extraction can't strictly re-derive the shipped table from scratch | Reproducibility | **Open problem** | Pinned seed + fingerprint | Pipeline is not yet fully re-runnable |

## Conclusion

"Verifiable," in this project, does not mean:

> **The system never gets an answer wrong.**

It means something more precise:

> **The system breaks an answer down into checkable facts, computations,
> relationships, and citations wherever it can, and states plainly how
> far machine verification currently reaches — and where it stops.**

The strongest guarantees today sit at the evidence-sourcing and citation-
tracing layer: financial numbers come from SQL/XBRL; relationships come
from structured edges carrying their own `source_text`; a number that
never appeared in the tool-call trace can be caught; a known class of
semantic-binding error already has a detector; and specific mechanisms can
each be independently verified by a deterministic probe.

The most visible remaining problem has shifted, gradually, from:

> **"Is this number made up?"**

toward:

> **"Are these real numbers being combined with the correct semantic
> relationship between them?"**

That shift is the next real frontier of reliability engineering for this
system.
