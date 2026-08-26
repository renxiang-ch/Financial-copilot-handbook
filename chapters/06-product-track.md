# 06 · Product Track (Optional)

**What you'll build**: nothing new to run — `start.ps1` (ch.01 §1.5) already
serves this. What you'll gain is the ability to explain *why* the What-if
view's ranking is split into two visually separate sections rather than one,
which is the single clearest place in the whole product where a UI decision
exists purely to prevent a specific, documented misreading.

Covers `frontend.py`, `views/*.py`, `dashboard.py`, and the `/dashboard/*`
half of `api.py`. Chat remains the primary entry point (`views/chat.py`);
this is the secondary one, reusing the same underlying data with zero
additional LLM cost — every function in `dashboard.py` is pure SQL.

## Three views, one shared principle

- **Company Exposure** (`views/exposure.py`) — for a chosen customer (default
  AAPL) and fiscal year, every named supplier ranked by dependency
  percentage, rendered as a radial cluster diagram. Each edge is clickable
  down to its `source_text` and SEC filing URL.
- **Supplier Deep Dive** (`views/supplier.py`) — one supplier's multi-year
  dependency trend on a chosen customer.
- **What-if** (`views/whatif.py`) — "if [customer] cuts orders by X%, which
  suppliers lose the most dollars" — computed from `revenue * pct/100 * cut`,
  pure SQL/pandas, no model call.

All three read the same `threshold_only` flag `dashboard.py` already carries
on every edge, and all three surface it the same way rather than letting a
threshold-only figure sit indistinguishable from an exact one.

## The What-if view's floor-estimate separation

The single clearest example in the product of a UI actively preventing a
specific, documented misreading.

A `threshold_only` edge (§3.4 — a disclosure that says "more than 10%" with
no exact figure) stores `revenue_pct = 10.0` as a floor, not an estimate of
the true value.

> **Skyworks' real Apple-dependency percentage**, for comparable years where
> an exact figure *is* available elsewhere, runs closer to **69%** — nearly
> seven times the stored floor. Ranking suppliers by projected dollar loss
> with that floor treated as a real number would silently and substantially
> understate a threshold-only supplier's true exposure, and rank it far too
> low.

`views/whatif.py` closes this by **splitting the ranking before displaying
it, not after**: `exact_results` (edges where `threshold_only` is false) form
the main ranked bar chart; `floor_results` are pulled into a visually
separate "⚠️ Floor estimates — real exposure likely higher, not ranked
against the above" section, explicitly not compared against the exact
figures. This is a UI decision made specifically because the underlying
number *cannot* be corrected computationally (the true percentage isn't in
the data) — so the interface itself carries the caveat a purely numeric
answer would lose. The same `threshold_only`-aware rendering (`>X%` instead
of a bare number, in tooltips and expander labels) is consistent across
`exposure.py`, `supplier.py`, and the chat view's graph-citation panel
(`views/chat.py`) — one flag, checked everywhere it needs to change how a
number is read, not just where it happens to have been added first.

## Checkpoint

No new command — with `start.ps1` running, open the What-if view, set
customer to AAPL, and cut orders by any percentage. **Pass criteria**:
Skyworks (SWKS) appears in a visually separate section below the main
ranked bar chart, labeled as a floor estimate, not comingled into the main
ranking — if it renders in the main chart's ranked list instead, something
about `threshold_only` handling has regressed, and that's a real bug to
chase, not a display quirk.
