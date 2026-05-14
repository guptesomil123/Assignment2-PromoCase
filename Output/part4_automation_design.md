# Part 4 — Automation Design

**Goal:** Run the full promo analysis workflow every month with minimal manual touch.

---

## Overview

The pipeline has five layers: sources, ingestion, object storage and warehouse,
transformation, and serving. Each is described below with tool choices, rationale,
and failure modes.

---

## 1. Data Ingestion

### Where does the data come from in production?

**Cymbiotika.com (Shopify — DTC)**
Shopify is the primary source and has the richest data: full order-level detail
including customer type, discount code, SKUs, net sales, and timestamps. In
production, a managed connector (Fivetran or Airbyte) syncs Shopify directly into
the data warehouse on an hourly schedule using incremental watermarks. This
eliminates custom API maintenance and handles schema changes automatically.

**Amazon (Seller Central)**
Amazon exposes a Selling Partner API (SP-API) that returns aggregated sales metrics
by ASIN. A scheduled job (daily, early morning) pulls settlement reports and order
metrics and lands them in object storage. Note: Amazon does not share individual
customer-level order data with third-party sellers — only aggregated sell-through
and settlement summaries are available.

**Target, Vitamin Shoppe, Ulta Beauty (Retail partners)**
Retail partners do not offer APIs. They operate on EDI (Electronic Data Interchange)
standards — structured flat files dropped to an SFTP server on a nightly or weekly
cadence. AWS Transfer Family hosts the SFTP endpoint. Each partner gets a dedicated
storage prefix and scoped credentials. A parsing job normalises the EDI or CSV files
into a consistent schema before they enter the warehouse.

Key implication for analysis: retail partner data is always 1–2 days behind DTC.
Vitamin Shoppe and Ulta report weekly, not daily, so day-level promo lift analysis
is only possible for the Shopify DTC channel.

---

## 2. Cleaning and Promo-Tagging Logic

### How is "is this a promo order" decided systematically?

The worst pattern is hardcoding promo definitions in analysis scripts — a date range
and a discount code buried inside a notebook. This breaks every new promo and
requires a code change to update.

**Production approach: a promo_config table**

A version-controlled configuration table in the warehouse stores all promo
definitions. Each row contains the promo name, start date, end date, eligible
discount codes, and promo type. Marketing Ops owns this table and inserts one row
when a new promo is added to the calendar. No code changes are required.

A transformation model joins every order to this table automatically. An order is
tagged as a promo order if its date falls within the window and its discount code
matches the eligible codes. All other orders are tagged as organic. The tagging
logic is transparent, auditable, and consistent across every report and dashboard.

**Cleaning rules applied in the transformation layer, not notebooks**

- Strip and uppercase discount codes; treat empty strings and nulls as equivalent
- Deduplicate order IDs, keeping the earliest occurrence
- Recompute AOV and UPT from net_sales and units rather than trusting source values
- Parse order dates with mixed-format tolerance; drop rows with unparseable dates
- Strip currency symbols from revenue fields before casting to numeric

---

## 3. Baseline Calculation

### Rolling or fixed window?

A fixed window (for example, always Feb 28 – Apr 24) goes stale and only works for
one specific promo date. A rolling 8-week trailing window parameterised by promo
start date works for any future event without modification.

**Recommended: rolling 8-week trailing window**

The baseline end date is always the day before the promo starts. The baseline start
date is always 56 days before that. Both dates are derived automatically from the
promo_config table. The analyst never manually enters a date range.

Day-of-week adjustment is built into the baseline model. Because promos often fall
on weekends (higher natural order volume), the model computes two baselines in
parallel — the full 8-week average and a weekend-only average using only Friday,
Saturday, and Sunday observations from the baseline window. Both are written to the
warehouse and surfaced in the dashboard.

---

## 4. Projection Model

### Where does it live? How are assumptions versioned?

The projection model has two components that should live in different places.

**Formula lives in the transformation layer, version-controlled in git**

The projection formula is a dbt model. dbt connects directly to Snowflake and reads
from the raw schema — the immutable tables loaded by Snowpipe. It writes clean,
transformed output to a separate analytics schema that Tableau or Looker queries.
Any change to the formula requires a pull request, code review, and merge before it
takes effect. The full history of every formula change is preserved. This prevents
accidental overwrites and ensures that historical projections are reproducible.

**Assumptions live in a forecast_assumptions table**

Growth multipliers, uplift scalars, and AOV adjustments are stored in a separate
table alongside the promo_config table. Each row captures the scenario (low, mid,
high), the multiplier values, who set them, and when. Finance or the analyst updates
this table before each promo. The projection model reads from it at runtime — no
code change is needed to update assumptions.

This separation means the formula is owned by the data team and changes slowly
through proper review, while the assumptions are owned by Finance and Marketing and
change frequently with no engineering involvement. If a Google Sheets integration is
not yet available, a locked Sheet tab with named ranges works as an interim
assumption store.

---

## 5. Output Destination

**Analysts — Tableau or Looker**
Connected directly to Snowflake. Both tools have first-class Snowflake connectors
and are the natural fit for an AWS-based stack. Tableau is the stronger choice if
the team prioritises self-serve visual analytics for non-technical stakeholders.
Looker is the stronger choice if the data team wants metric definitions centralised
in code (LookML), ensuring every report uses the same numbers regardless of who
built the dashboard. Power BI is a viable alternative only if the organisation is
already on a Microsoft stack — it works across clouds but adds friction in an
AWS-native environment.

**Finance and Marketing — Google Sheets**
Non-technical stakeholders prefer Sheets over BI tools. At the end of each pipeline
run, an automated step writes the promo summary table to a locked Sheet tab. 
Stakeholders see a live, auto-refreshing view without needing BI tool access. A
separate tab exposes the forecast_assumptions table so Finance can update multipliers
directly.

**Everyone — Slack**
Two channels serve different purposes. A monitoring channel receives alerts when
something breaks or looks anomalous — data quality failures, row count spikes, null
rate thresholds breached. An operations channel receives a brief summary message when
the pipeline completes successfully, including headline promo numbers.

---

## 6. Monitoring

### How do you know when it breaks?

Pipelines fail silently more often than they fail loudly. Three layers of monitoring
catch different failure modes.

**Layer 1 — Schema and logic tests, run on every pipeline execution**

The transformation layer runs automated tests on every model after each build.
Tests check that order IDs are unique and non-null, order dates are non-null,
net_sales are non-negative, customer_type values are within the expected set, and
discount codes match known patterns. If any test fails, the pipeline halts and sends
an alert. No silent bad data reaches the dashboard or the Sheets output.

**Layer 2 — Row count and freshness checks**

After each ingestion run, an automated check compares yesterday's order count
against a rolling expected range. If the count is suspiciously low (possible
ingestion failure) or suspiciously high (possible double-load), an alert fires. A
freshness check also verifies that each source table has received new data within
the expected time window, catching cases where a source feed goes silent without
throwing an error.

**Layer 3 — Null rate monitoring**

For key columns (order_date, net_sales, customer_type, discount_code), the pipeline
tracks the null rate on each run. If the null rate on any column exceeds a defined
threshold, an alert fires. This catches upstream data quality degradation — for
example, if Shopify starts sending blank discount_code fields — before it corrupts
downstream metrics.

**Alert routing**

- Critical failures (test failures, pipeline halted) → monitoring Slack channel, immediate
- Warnings (row count anomalies, elevated null rates) → same channel, lower urgency
- Successful run with headline stats → operations Slack channel, informational

---

## Summary — Full Monthly Run

When a new promo is added to the calendar:

1. Marketing Ops inserts one row into the promo_config table with dates and codes
2. Analyst inserts assumption rows into forecast_assumptions (low, mid, high)
3. The Airflow DAG runs automatically and orchestrates the entire pipeline end to end:
   - Step 1: senses ingestion completion — waits for Fivetran sync, Lambda pull, and SFTP file arrival before proceeding
   - Step 2: confirms Snowpipe has loaded all files into Snowflake's raw schema
   - Step 3: triggers dbt run — models clean, tag, baseline, lift, and projections
   - Step 4: runs dbt tests — pipeline halts and alerts if any test fails
   - Step 5: exports outputs — Google Sheets refreshed, Slack summary posted
4. Analyst reviews the dashboard for anomalies and checks monitoring alerts
5. Post-event, forecast_assumptions are updated with actuals for model calibration

**Total manual touch per promo: approximately 15 minutes.**
Everything else is automated and repeatable for any future event.
