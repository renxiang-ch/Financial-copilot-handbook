# 05 · Evaluation

Covers `eval/harness.py`, `eval/harness_tier3.py`, `eval/harness_router.py`,
five `eval/probe_*.py` modules, and `data/datasets/*.json` (six sets — check
the directory, an earlier draft of this mapping missed two of them).

## 5.1 The eval sets

<!-- TODO: for each of the six sets in data/datasets/, state: question count
     (check the file, don't reuse an old count -- these have all changed),
     what it tests, why it exists. Include eval_set.json's RETIREMENT
     mechanism as a teaching point: a frozen set protects against tuning to
     the test, but cannot stop a question's premise from becoming false as the
     database changes -- retirement is how that tension is resolved without
     breaking the freeze discipline. -->

## 5.2 How the harness scores an answer

<!-- TODO: numeric tolerance (SI-scale + tolerance_abs), input-value checking,
     the `grounded` scoring type (outcome-scored, not tool-path-scored -- and
     WHY: a documented false negative where the harness only credited
     retrieve_text hits and scored a better graph_query answer as a miss). -->

## 5.3 Zero-cost deterministic probes

<!-- TODO: this section did not exist in the earlier draft of this mapping and
     should not be skipped -- it is the strongest methodological content in
     the project. probe_router (regex vs slots), probe_retrieval /
     probe_truncation (chunking defects), probe_year_scope, probe_clarify.
     The throughline: when an eval set can't distinguish signal from noise
     (retrieval accuracy's 25-62.5% historical band is wider than most real
     changes), build a deterministic zero-LLM-cost probe that isolates the
     specific mechanism instead. -->

## 5.4 Running an evaluation and reading the result

<!-- TODO: the db_fingerprint contract -- point to appendix b for the
     expected-value table. Cost/latency reporting. -->

## 5.5 Ablations

<!-- TODO: harness_tier3 --no-graph (baseline naive-RAG vs graph-augmented),
     harness_router before/after, the multi-turn set's --repeat / --no-inherit
     flags and why a nine-turn set needed five repeated runs before a real
     effect was distinguishable from noise. -->
