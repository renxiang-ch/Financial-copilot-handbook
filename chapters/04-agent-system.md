# 04 · Agent System

Covers all 10 modules in `src/copilot/agent/`. This chapter is bigger than the
others — the earlier draft of this handbook covered only 4 of the 10 files;
the other six are where most of the actual engineering decisions live.

## 4.1 The loop and the tools

<!-- TODO: agent.py's ask()/_ask_openai(), tools.py's five tools. Note:
     _ask_anthropic no longer exists -- check the current file before writing
     this, don't assume from an old summary. -->

## 4.2 Slots — one parser, several consumers

<!-- TODO: slots.py. Deterministic regex + a company index read FROM THE
     DATABASE (not hardcoded). Walk through extract_slots() on one example
     question end to end. The `unaccounted` channel -- why a parser needs a
     way to say "I didn't fully understand this" -- is the single best
     teaching point in this file. -->

## 4.3 Routing — by relationship direction, not by keyword

<!-- TODO: route_question() in agent.py. The regex-router-vs-slots-router
     history is a good worked example: measured 38.9% on phrasings drawn from
     real failures while its own hand-written harness reported 100%, because
     the harness's questions were written by reading the regex. Show
     probe_router.py as the fix (deterministic, zero-cost). -->

## 4.4 Multi-turn: inheritance and clarification

<!-- TODO: conversation.py (what a follow-up inherits, and why company
     inheritance is substitution-into-the-pronoun's-position rather than list
     merging) + clarify.py (when the system asks the user vs. when asking
     would be wrong -- the three "uncertain" signals that do NOT trigger a
     question, and why). -->

## 4.5 Verification — checking groundedness, not correctness

<!-- TODO: grounding.py's five checks + authority.py's formula-provenance
     classifier. State plainly what each does NOT catch (grounding.py: correct
     arithmetic over the wrong inputs, in general; authority.py: whether a
     formula is the RIGHT one, only whether it has a known source). This
     boundary is as important to teach as the mechanism itself. -->

## 4.6 Model routing

<!-- TODO: model_router.py. Short file, but the decision record in its
     comments (why there is no automatic escalation to a stronger model, after
     checking what mainstream agent frameworks actually do) is worth
     reproducing near-verbatim -- it is a rare case of a negative result
     written down with its reasoning intact. -->
