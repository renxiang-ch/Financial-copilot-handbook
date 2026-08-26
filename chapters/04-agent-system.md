# 04 · Agent System

Covers all modules in `src/copilot/agent/`: `agent.py`, `tools.py`,
`slots.py`, `grounding.py`, `authority.py`, `clarify.py`, `conversation.py`,
`provenance.py`, `model_router.py`. This chapter is bigger than the others —
most of the actual engineering decisions in this project live here, and six
of these nine files did not exist in an earlier version of this system.

## 4.1 The loop and the tools

`agent.py::ask(question, model=None)` is the entry point. It is a
hand-written tool-calling loop, not built on an agent framework: send the
conversation + tool schemas to the model, execute whatever it calls (in
parallel within a round via a thread pool, `_run_tool`), append results,
repeat until a final answer or a 10-round circuit breaker fires. The only
model-dispatch function is `_ask_openai()` — there is no separate Anthropic
code path in the current codebase; an earlier `_ask_anthropic()` existed and
was removed. Check `agent.py` directly before assuming otherwise, since this
is exactly the kind of detail that goes stale between a summary and the
code.

`tools.py` exposes five tools:

| Tool | Does |
|---|---|
| `query_financials(ticker, metric, fiscal_year)` | Direct SQL lookup against `financial_facts` |
| `list_metrics(ticker)` | Discover which metrics/years exist for a company before querying |
| `compute(expression, variables)` | Sandboxed arithmetic (AST-whitelisted, no `__import__`, exponent depth capped) — the only place a number is allowed to be derived rather than looked up |
| `retrieve_text(query, ticker, fiscal_year)` | Hybrid retrieval over filing prose/tables (§3.3) |
| `graph_query(customer, supplier, fiscal_year, depth)` | Recursive-CTE traversal over `supply_edges` |

All three ticker-taking tools (`query_financials`, `list_metrics`,
`graph_query`) route ticker resolution through a shared helper
(`_resolve_ticker`) that distinguishes "this ticker doesn't exist" from
"this ticker exists but has no row for this query" and offers a
`difflib`-based correction (`SKWS` → `SWKS`) rather than reporting a
misspelling as a fact about the world — see ch.07 for the specific failure
this closed.

## 4.2 Slots — one parser, several consumers

`slots.py::extract_slots(question, carry=None)` is a single deterministic
parser (regex + a company index read from the live `companies` table, never
hardcoded) that three different consumers rely on: the router (§4.3), the
multi-turn inheritance mechanism (§4.4), and year-scoped retrieval (§3.3).
Building it once and sharing it, rather than three separate ad-hoc parsers,
is a deliberate choice — a company-name regex fixed in one place fixes it
for routing, inheritance, and retrieval simultaneously.

Worked example — `find_companies()` finds every company mentioned;
`find_metric()` classifies the metric family being asked about;
`relation(text)` determines which two companies are in a supplier↔customer
relationship and in which direction (critical for routing, §4.3);
`unaccounted(text)` is the single best teaching point in this file: rather
than forcing every question into a confident slot assignment, it returns
the list of things the parser noticed but could not classify —
a channel for the parser to say "I didn't fully understand this" instead of
guessing silently. Downstream consumers (the router especially) treat a
non-empty `unaccounted` as a reason to hold back from an irreversible action
(like refusing outright) rather than as noise to ignore.

## 4.3 Routing — by relationship direction, not by keyword

`agent.py::route_question(question, carry=None)` classifies a question
before the agent runs at all: some categories are force-routed to a specific
tool (skipping the model's own tool-selection judgment), and one category
(customer-side procurement-share questions — structurally unanswerable,
since these companies' 10-Ks report supplier-side concentration, never the
customer's side) is refused immediately at zero tool calls. The router uses
`slots.relation()`'s direction, not keyword matching — "what percentage of
Cirrus Logic's revenue comes from Apple" and "how much of Apple's component
sourcing is Cirrus Logic" use almost the same vocabulary but are opposite,
and only one is answerable from this data.

**A worked example in measurement discipline, not just mechanism**: an
earlier keyword-regex version of this router was validated by its own
hand-written 12-question test harness at 100% tool-selection accuracy. A
later, independently-built 18-question probe — written from real observed
failure phrasings rather than by reading the regex being tested — measured
the *same* regex router at **38.9%**. The harness wasn't lying; its
questions had unconsciously been written by someone who already knew what
the regex matched, so it could only ever confirm the regex, never challenge
it. The fix (rebuilding routing on `slots.relation()`'s semantic direction
rather than keywords) brought the 18-question probe to 100% with zero false
refusals — a real fix, but the more durable lesson for anyone extending this
router is upstream of that fix: **a test written by reading the
implementation cannot measure the implementation**, and this project's own
history contains a concrete, quantified instance of that trap.

## 4.4 Multi-turn: inheritance and clarification

`conversation.py` manages what a follow-up question inherits from prior
turns. `carried_slots(history)` extracts the previous turn's slots;
`active_context_block(slots)` renders them into a short block injected
*after* the conversation history and *before* the new question — so a
single-turn question's prompt is byte-identical to before this feature
existed (nothing is injected when there's nothing to carry), preserving
prompt-cache prefix stability for the common case.

**Company inheritance is substitution into the pronoun's position, not list
merging** — deliberately. "How dependent is it on Samsung?" following a
question about Cirrus Logic should resolve "it" → Cirrus Logic and ask about
*that* relationship; naively appending Cirrus Logic to a company list built
from "Samsung" would produce a relationship in the wrong direction, since
`slots.relation()`'s direction logic is positional. The pronoun is replaced
by the carried company, the sentence is re-parsed from scratch, and the
substitution is discarded if re-parsing doesn't produce a valid relation —
never trusted blindly.

`clarify.py::clarification_for(question, slots, carry)` decides when to ask
the user rather than guess. It is deliberately narrow: it only fires when
**multiple readings exist and every reading is separately answerable** (an
ambiguous metric family like "profit", or a pronoun with more than one
possible antecedent) — `_ambiguous_metric()` and the pronoun-candidate path
in `clarification_for()` are the two live triggers. It does **not** fire for
a misspelled ticker (only one reading is correct — the tool layer's
`difflib` correction handles it, §4.1), a reversed relationship direction
(same reason), or a segment/geography question the system can't answer under
*any* reading (asking a menu of options here would falsely imply one of them
is answerable). Every option a clarification offers carries `allow_other:
True` — the system's two guessed readings are not assumed to exhaust the
user's actual intent.

## 4.5 Verification — checking groundedness, not correctness

Two modules run *after* the agent has already produced an answer, checking
it rather than helping produce it — a deliberate separation, so a check
can't accidentally become an assumption the answer-generation step leans on.

`grounding.py::verify_answer(steps, answer, question)` checks whether every
number and accession cited in the final answer text actually traces back to
a tool call in the trace — catching a model that states a plausible-looking
number nothing in the trace ever produced. It explicitly does **not** catch
correct arithmetic performed over the wrong inputs in general — a number
that is grounded (it did come from a real tool call) can still be the wrong
number for the question asked. One specific sub-class of that general gap
*is* caught (`_binding_mismatches` — a supply-edge percentage that's
supplier-scoped, applied to the wrong company's revenue or the wrong fiscal
year), because that specific pattern is checkable without knowing what the
question actually meant; the general case is not, and the module's own
tests pin this boundary explicitly (`test_the_wrong_metric_is_still_not_
caught`) so the gap stays visible rather than silently widening.

`authority.py::classify(step, steps, question)` answers a narrower question
about `compute` calls specifically: not "is this formula correct" (that
would require enumerating every valid financial formula, an unbounded set),
but "where did this formula come from" — three buckets: `question` (the
question itself specified the arithmetic, e.g. "if orders drop 20%"),
`registry` (the label combination matches one of ~10 formulas observed
repeatedly correct in this project's own trace history — gross margin, YoY,
CAGR, order-cut impact, etc.), or `none` (the complement, unbounded, never
enumerated). A `none` classification is not treated as an error — it is
surfaced to the reader as an unsourced derivation, since a syntactically
correct computation can still be the wrong concept (the project's own
history has a case of `total_equity / (total_assets - total_debt)` being
computed, confidently, for a question about book value per share — real
arithmetic, wrong formula, and exactly the class this classifier exists to
flag rather than silently pass).

`provenance.py::build_provenance(steps, answer)` collects every accession
number and figure referenced across *all* tool calls (not just
`query_financials`, an earlier version's bug) into one canonical citation
list, resolving EDGAR's two accession-number formats (hyphenated in prose,
unhyphenated in URLs) to the same identity so a citation isn't silently
missed because of which form appeared in the trace.

## 4.6 Model routing

`model_router.py` is a five-line file with a long comment above it — the
comment is the point, and is worth reading in full rather than summarizing,
so it is reproduced here near-verbatim rather than paraphrased:

An earlier version routed by question category to a cheap or a strong
model tier, and escalated to the strong tier when the cheap one showed
signs of having failed. Both halves were removed, for measured reasons.
**Tiering** was removed because the frozen 33-question eval set gave
*identical* scores on every deterministic metric between `gpt-4o` and
`gpt-4o-mini` — at 16× the cost for `gpt-4o`, with no change in any number.
The honest reading is narrower than "the models are equivalent": the eval
set is saturated, so it has no power to resolve a real difference either
way — and a tiering table where every category falls through to the same
tier is dead configuration, the same hardcoded-list-drifts-from-data defect
this project keeps finding elsewhere (ch.07). **Escalation** was removed
after checking what mainstream agent frameworks actually do at this
decision point: LangChain's model-fallback middleware, LangGraph's tool
retry, LiteLLM's fallbacks, and the OpenAI Agents SDK all switch models on
*exceptions* (429s, timeouts, context overruns) and never on judged answer
quality. Quality problems get a different response everywhere they're
handled at all — Pydantic AI feeds a validation failure back to the *same*
model, and the OpenAI Agents SDK trips a guardrail and halts rather than
retrying with a bigger model. Nobody in the surveyed frameworks maps "this
answer looks wrong" to "pay more" — and the reasoning generalizes: a
stronger model is a fix for a model that could not do the task; it is not a
fix for a fabricated figure, because a stronger model fabricates more
fluently, not less. What this system does instead is verify (§4.5) —
detection is the part worth building; automatic escalation was a guess
dressed as a policy. What remains is `select_model(requested)`: an explicit
model choice is always honored; absent one, `DEFAULT_MODEL = "gpt-4o-mini"`.
