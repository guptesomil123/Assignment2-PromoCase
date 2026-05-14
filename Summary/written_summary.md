# Cymbiotika Take-Home #2 — Written Summary

**Analyst:** Somil Gupte
**Date:** 13 May 2025
**Event:** SPRING20 — 20% off sitewide, Apr 25–27 2025

---

## Part 1 — Data Cleaning

**Headline numbers:**
Raw file contained 5,678 orders across Feb–Apr 2025. After deduplication,
date parsing, and null removal, the cleaned dataset retained all usable rows
with no material data loss.

**Key cleaning decisions:**
- Empty strings, `"nan"`, and `"None"` in `discount_code` were all mapped to
  null — treating them differently would have caused false negatives in every
  downstream promo filter.
- `aov` and `upt` were recomputed from `net_sales` and `units` rather than
  trusted from source. Derived columns in raw exports frequently drift from
  their source fields; recomputing ensures consistency.
- Duplicate `order_id` rows were deduplicated keeping the first occurrence.
  Later rows in a re-export are typically corrections, but without a
  `modified_at` timestamp the conservative choice is to keep the earliest.
- Promo definition uses the intersection of date window AND discount code —
  not date alone (would include organic orders) and not code alone (would
  miss timing constraints). This is the most defensible definition for lift
  analysis.

**SPRING20 promo summary:**
357 orders | $35,627 net sales | $99.80 AOV | 2.11 UPT

**What I'd do with more time:**
Obtain a line-items table (one row per SKU per order) from Shopify. The
current `skus` column is pipe-delimited at the order level, which requires
the equal-split approximation used in Part 6. Line-item data would eliminate
that assumption entirely.

---

## Part 2 — 8-Week Trailing Baseline & Lift

**Headline numbers:**

| Metric | Baseline (daily avg) | Promo (daily avg) | Lift |
|---|---|---|---|
| Orders | 70.8 | 119.0 | +68% |
| Net sales ($) | $8,371 | $11,876 | +42% |
| AOV ($) | $118.31 | $99.80 | −16% |
| UPT | 2.05 | 2.11 | +3% |

**Key decisions:**
- Baseline uses all orders in the window, including other discount codes
  (WELCOME10, FRIEND10, SUB15). Stripping them would artificially depress
  the baseline and inflate lift — that is not what "normal" looked like.
- A weekend-adjusted baseline was also computed (Fri/Sat/Sun only) because
  Apr 25–27 was a Friday–Sunday. The raw +68% order lift partially reflects
  the natural weekend effect. The weekend-adjusted lift is the more
  defensible figure for measuring the true promo contribution.
- Revenue lift (+42%) is lower than order lift (+68%) because the 20%
  discount compressed AOV by 16%. This is expected and not a data issue.

**Caveats documented:**
- Day-of-week mix: weekend days have naturally higher order volume
- Pull-forward effect: orders may have been deferred from the days before
  the promo, borrowing from adjacent demand rather than generating new demand
- Attribution: not all Apr 25–27 orders used SPRING20 — organic orders ran
  alongside promo orders. Both views (code-only and full window) are reported.

**What I'd do with more time:**
Run a difference-in-differences analysis using a control group (e.g. customers
who did not receive the promo email) to isolate truly incremental orders from
pull-forward and organic demand.

---

## Part 3 — Simple Projection Model

**Headline numbers (mid-case):**

| Scenario | Projected orders | Projected revenue | Growth rate |
|---|---|---|---|
| Low | ~312 | ~$28,400 | 1.10x |
| Mid | ~355 | ~$33,200 | 1.18x |
| High | ~390 | ~$37,800 | 1.25x |

**Key assumptions and justification:**
- **YoY growth 1.18x (mid):** Cymbiotika grew total revenue from $100M (2024)
  to $150M (2025) = 1.50x overall. Retail expansion (Target, Vitamin Shoppe)
  absorbed a meaningful share of that growth, putting DTC-specific growth at
  ~1.18–1.25x. The 1.18x assignment input is therefore conservative but
  defensible. The $250M LinkedIn projection proved overstated (actual: $150M),
  which is why the high case is capped at 1.25x rather than 1.30x.
- **Promo AOV adjustment (0.84x mid):** Observed this year: TY promo AOV was
  $99.80 vs baseline $118.31 — a 15.7% compression, in line with expectations
  for a 20% sitewide discount partially offset by larger baskets.
- **Revenue is not directly anchored to LY** — only LY unit data exists in
  `last_year_promo_skus.csv`. Revenue is derived from projected orders × TY
  AOV. This is the weakest link in the model and is flagged explicitly.

**What I'd do with more time:**
Obtain LY order-level revenue data to anchor the revenue projection directly
rather than deriving it from TY AOV assumptions.

---

## Part 4 — Automation Design

**Architecture summary:**
Five-layer pipeline: Shopify (Fivetran) + Amazon (Lambda/SP-API) + retail
partners (AWS Transfer Family SFTP) → S3 raw bucket → Snowpipe → Snowflake
→ dbt (reads raw schema, writes analytics schema) → Tableau/Looker +
Google Sheets + Slack. Airflow (MWAA) orchestrates all steps from ingestion
sensing through export.

**Key design decisions:**
- **promo_config table:** Promo definitions live in a versioned database table,
  not hardcoded in scripts. Adding a new promo = one row inserted by Marketing
  Ops. Zero code changes required.
- **Airflow as full-pipeline orchestrator:** Airflow senses ingestion
  completion, triggers dbt, runs tests, and pushes outputs. It is not a peer
  component sitting alongside dbt — it wraps the entire pipeline.
- **dbt reads from Snowflake raw schema:** Raw data is never modified. dbt
  writes to a separate analytics schema. A dbt bug can never corrupt source
  data.
- **Retail partners use SFTP/EDI, not APIs:** Target, Vitamin Shoppe, and Ulta
  operate on EDI standards with nightly/weekly file drops. Day-level granularity
  is only available for the DTC Shopify channel.
- **Tableau or Looker over Power BI:** Cymbiotika runs on AWS. Power BI is
  Microsoft-native and adds cross-cloud friction without meaningful benefit.

**What I'd do with more time:**
Build a `forecast_assumptions` table with a Google Sheets front-end so Finance
can update growth multipliers directly before each promo without needing
data team involvement.

---

## Part 5 — Forecasting Model

**Headline numbers:**

| Scenario | Total orders | Net sales | AOV |
|---|---|---|---|
| Low (1.10x) | ~312 | ~$24,900 | ~$79.80 |
| Mid (1.18x) | ~355 | ~$28,400 | ~$79.90 |
| High (1.25x) | ~390 | ~$31,200 | ~$80.00 |

**Model architecture — two-anchor blended forecast:**
Rather than a single multiplier, the model cross-checks two independent signals
and blends them 60/40:

- **Anchor A (60%):** LY SPRING20 event performance × YoY growth × uplift
  scalar. Captures event-specific demand patterns.
- **Anchor B (40%):** TY 8-week baseline × LY uplift factor × uplift scalar.
  Captures current business momentum.

Blending guards against over-relying on either signal. If LY had an anomaly,
Anchor A would mislead. If TY baseline drifted from trend, Anchor B would
mislead. The blend dampens both risks.

**Sensitivity analysis — which assumption moves the forecast most:**
The YoY growth rate is the single most impactful assumption. A ±10% change
in the growth rate swings total revenue by approximately $2,000–$2,500.
This is the number Finance should pressure-test hardest before committing
to inventory or staffing.

**What I'd do with more time:**
Back-test the model against 2–3 prior promos to establish whether the
two-anchor blend consistently outperforms a single-multiplier approach.
Without back-testing, the 60/40 blend weight is a reasoned judgment
rather than a data-derived parameter.

---

## Part 6 — SKU-Level Forecasting

**Headline numbers (mid-case):**
- 4,206 units projected during 3-day promo week
- 851 units projected in post-promo week (pull-forward hangover)
- 5,057 total 2-week unit requirement

**Critical finding — all 9 in-promo SKUs are at stockout risk:**
Every SKU's current inventory is below even the low-case 2-week forecast.
Aggregate mid-case shortfall: 2,133 units. The three most exposed are
CYM-MAG-30 (389 units short, 6 days cover), CYM-COL-30 (299 units short,
6 days cover), and CYM-VTC-30 (292 units short, 6 days cover).

**Recommended actions for Supply Chain:**
1. Expedite reorder on all in-promo SKUs, prioritised by gap size
2. Cap purchase quantity at 2 units per customer to extend inventory
   coverage across the full event window
3. Consider removing the lowest-inventory SKUs (CYM-MAG-30, CYM-COL-30,
   CYM-VTC-30) from promo eligibility if reorder cannot arrive before launch

**SKU tiering approach:**
SKUs were grouped into three tiers by LY uplift magnitude rather than
applying per-SKU multipliers. Per-SKU multipliers on a single year of data
are noise-amplifying — a tier approach is more robust and still captures the
meaningful variance between hero SKUs (≥2.5x uplift) and staples (<1.8x).
All 9 in-promo SKUs landed in the High tier based on LY data, which is
consistent with Cymbiotika's positioning — customers actively seek these
products during sale events.

**Post-promo dip:**
Demand is forecast to drop ~80% in the week following the event as
pull-forward demand normalises. Do not replenish to promo-week inventory
levels mid-event. Top 3 revenue contributors — CYM-GLU-30 (18%),
CYM-MAG-30 (15%), CYM-PRO-30 (14%) — account for 47% of promo revenue
and should be prioritised for inventory depth and reorder speed.

**Key assumption documented:**
Units per SKU were approximated as total order units ÷ number of SKUs
in that order, because only order-level data is available (no line-items
table). The equal-split approximation is directionally correct given that
most orders show matching unit and SKU counts.

**What I'd do with more time:**
Obtain line-item data from Shopify to eliminate the equal-split assumption.
Additionally, model stockout probability as a distribution rather than a
deterministic flag — accounting for demand variance day-by-day within the
promo window would give Supply Chain a cleaner reorder quantity recommendation.

---

## Overall — Intellectual Honesty Notes

Several inputs in this assignment were given rather than derived from data:

- **1.18x YoY growth rate** — unverifiable from the provided dataset (no 2024
  order-level data). Accepted as mid-case but interrogated against external
  revenue data (Cymbiotika $100M→$150M actual 2025 growth). The rate is
  conservative and defensible for DTC once retail channel growth is separated.
- **75% LY volume assumption** — used as instructed to derive LY anchors.
  All metrics derived from this assumption are clearly labelled as derived
  rather than observed.
- **Revenue projections** — not directly anchored to LY revenue (no LY
  revenue data available). Derived from projected orders × TY AOV. This
  introduces an additional assumption layer that is documented throughout.

Where numbers felt arbitrary, they were flagged rather than presented as
precise. The goal was a model Finance and Supply Chain can actually plan
against — which requires knowing where the numbers are solid and where
they are estimates.
