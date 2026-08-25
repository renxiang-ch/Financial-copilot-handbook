# 07 · Limitations & Pitfalls

No single owning file — collected from every chapter plus the project's own
engineering log (`CLAUDE.md` in repo A).

## A note on structure

An earlier draft of this handbook proposed a single sentence here: "every data
asset the pipeline produces has a mechanism in evaluation guarding it." **Do
not write that sentence.** It is not true as stated — guard strength varies
enormously (numeric questions have been saturated at 100% for months and
carry little signal; retrieval accuracy's historical noise band, 25–62.5%, is
wider than almost any real improvement). Replace it with a table:

| Data asset | What guards it | How strong is the guard |
|---|---|---|
<!-- TODO: fill in per-asset, honestly, including the weak ones -->

## The recurring defect class

<!-- TODO: "a hardcoded inventory drifts from the data it describes" has
     recurred roughly ten times across this project's history (see repo A's
     CLAUDE.md and docs/simplification-audit.md for the full list) -- from a
     supplier dropdown built on a color table to a foreign-key-less table
     definition that let 33 provenance pointers rot silently. Walk through two
     or three concrete instances with root cause and fix. This is the single
     most teachable failure pattern in the whole project. -->

## Known, unfixed, currently

<!-- TODO: pull the CURRENT state at handbook-writing time from repo A's
     README "Known Limitations" section -- do not copy this list from
     memory, it changes. As of the last check it included: relationship-
     direction inversion in supply-chain answers (~10% residual, detector but
     no fix), formula invention on derived metrics with missing inputs
     (recorded and shown to the user rather than refused, by design), and the
     LLM-judge same-model self-evaluation bias. -->
