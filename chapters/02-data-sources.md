# 02 · Data Sources

**What you'll build**: nothing yet, deliberately — this chapter is the
background knowledge ch.03's pipeline assumes you already have. No code
file owns it. Skip it if you already know what a customer-concentration
disclosure is and why ASC 280 makes it extractable; come back if ch.03's
whitelist or the company-tier table stops making sense.

## SEC EDGAR: two different APIs for two different kinds of fact

- **XBRL `companyfacts` / `frames` API** — every US public company tags its
  financial statement line items (revenue, net income, R&D, etc.) in a
  standardized taxonomy (`us-gaap:...`) as part of every 10-K/10-Q filing.
  `companyfacts` returns, per company, every tagged fact across every filing
  it has ever made — hundreds of raw tags, most irrelevant to this project.
  This is the source for `financial_facts`: structured, machine-readable,
  no PDF/table parsing required, at the cost of a filtering problem (see
  ch.03 §3.1's whitelist).
- **Full-text HTML filings** — the 10-K document itself, for anything with
  no XBRL equivalent: Risk Factors, MD&A prose, and — critically for this
  project — customer-concentration disclosures. This is the source for
  `text_chunks` and, downstream, `supply_edges`.

## What a customer-concentration disclosure is, and why it's extractable

**ASC 280-10-50-42** (the FASB's segment-reporting standard) requires a
company to disclose any single customer that accounts for ≥10% of its
revenue. This is why supply-chain relationships are extractable at all: it
is not sentiment analysis or inference over vague language, it is a company
being legally required to name a concentration and (usually) state a
percentage. The disclosure comes in one of three forms, all present in this
project's corpus:

- **Named, exact** — "Apple Inc. accounted for 87% of total revenue" (Cirrus
  Logic's typical form).
- **Named, threshold-only** — "one customer accounted for more than ten
  percent" with no exact figure (Skyworks' typical form for Apple in some
  years) — extracted with `threshold_only = true`, a real floor rather than
  a fabricated point estimate.
- **Unnamed** — "our largest customer accounted for X%" with no name given.
  Present in the corpus but filtered out of every product-facing query
  (`disclosure_status = 'named'` is a hard `WHERE` clause everywhere the
  product reads `supply_edges`).

## The company cluster

15 companies, defined in `pipeline/companies.py`, tiered by ASC 280 status:

| Tier | Companies | Why |
|---|---|---|
| Hub | AAPL | The customer every other tier's disclosures are about |
| Positive (named ≥10% Apple concentration) | CRUS, QRVO, SWKS, AVGO, QCOM | The core supply-chain-graph evidence — these are the companies whose 10-Ks name Apple explicitly |
| Negative (diversified, no single named customer ≥10%) | GLW, ADI, TXN, MCHP, ON, LRCX | Deliberately included as a contrast set — proves the pipeline correctly finds *nothing* rather than always finding a relationship |
| EMS / connector (structural inference only, no disclosed customer name) | APH, JBL, SANM | Present in filings' customer lists via other structural evidence, without a named-disclosure ground truth to validate against |

**Constraint: US-listed 10-K filers only.** Foreign private issuers (TSMC,
Foxconn) file Form 20-F, a different disclosure regime this pipeline does
not parse — excluded by design, not by oversight.

## Current scale

| | |
|---|---|
| Companies | 15 |
| Filings | 148 |
| Financial facts | 38,966, across 24 canonical labels (full list below) |
| Text chunks | 16,342 (11,990 prose, 4,352 table), all embedded |
| Supply-chain edges | 128 total, 103 `disclosure_status = 'named'` |

The 24 canonical labels, pulled with `SELECT DISTINCT label FROM
financial_facts WHERE form='10-K'`:

`Revenue` · `COGS` · `GrossProfit` · `OperatingIncome` · `NetIncome` ·
`EPS_Basic` · `EPS_Diluted` · `R&D` · `TotalAssets` · `LongTermDebt` ·
`TotalDebt` · `TotalEquity` · `TotalEquityInclNCI` · `CurrentAssets` ·
`CurrentLiabilities` · `Inventory` · `PP&E` · `CapEx` · `D&A` ·
`D&A_Component` · `OperatingCashFlow` · `InterestExpense` ·
`InterestExpenseOnDebt` · `IncomeTaxExpense`

> **Before trusting any number on this page**: it was checked live against
> the database on 2026-08-25 and will have moved since. Re-run
> `copilot.eval.harness` and compare its `db_fingerprint` output rather than
> trusting this page past that date — and never hardcode the label list
> above into anything downstream that needs to stay current. A fixed list
> silently drifting from the live data it was copied from is the single most
> recurring defect class in this project's history (ch.07).

## Checkpoint

No command here — this chapter has no artifact of its own to run. The
self-check is conceptual: you should be able to explain, without looking
back, why a `table_only` fact (§ above) can't just be added to
`financial_facts` with a slightly wider XBRL-tag whitelist, and why an
`unnamed` disclosure (same section) is excluded from `graph_query` results
rather than shown with a caveat. Both answers are load-bearing for
ch.03's design — if either feels shaky, that's the signal to re-read this
chapter rather than push forward into the pipeline that assumes it.
