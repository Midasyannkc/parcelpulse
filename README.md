# ParcelPulse

**Delivery performance and customer satisfaction analytics, built on 99,441 real marketplace
orders.** The project answers one decision: *should the promised delivery date be padded, and by
how much?* It is built the way a senior data analyst would ship it: SQL for the certified
numbers, R for the statistics, an Excel decision model, a Tableau extract and dashboard spec, an
Omni semantic model, and a one-page decision memo.

Built against the pattern in recent Senior Data Analyst postings (SQL, R, Tableau or Power BI,
advanced Excel, root-cause analysis, experimentation, metric definitions, Omni-style semantic
layers), with the Walmart Advanced Analytics last-mile posting as the closest match.

## The scenario (SLO)

Every Monday, operations leadership reads a delivery-health report. Two promises must hold:

1. **Revenue ties to finance to the cent** in every month.
2. **The late rate counts every parcel we promised**, including the ones that never arrived.

Three incidents broke those promises or the conclusions drawn from them. All three were
reproduced, fixed and pinned by tests (`docs/incidents.md`).

## Real data (and where the file is)

**Brazilian E-Commerce Public Dataset by Olist**, published by Olist on Kaggle
(`olistbr/brazilian-ecommerce`, CC BY-NC-SA 4.0): 99,441 orders from 2016-09-04 to 2018-10-17.

* The data files are committed in [`data/bronze/`](data/bronze/), for example
  [`olist_orders_dataset.csv`](data/bronze/olist_orders_dataset.csv) and
  [`olist_order_items_dataset.csv`](data/bronze/olist_order_items_dataset.csv).
* Source, row counts and the disclosed substitution are in [`data/bronze/SOURCE.md`](data/bronze/SOURCE.md).
  **Kaggle was unreachable, so the files came from a public GitHub redistribution** and were
  checked against widely published totals (99,441 orders; 13,591,643.70 BRL item price) rather than
  byte-compared with Kaggle.

## What it found

| Finding | Number |
|---|---|
| A naive multi-join overstated revenue | +676,591.98 BRL (+5.01%); fixed version ties to the cent in 24 of 24 months |
| True late rate, counting parcels that never arrived | **9.17%**, not 8.11% (1,106 stuck parcels) |
| Worst region | Northeast, 16.2% late (95% CI 15.5 to 17.0), median 17.6 days to deliver |
| Review gap, late vs on time, all reviews | 1.74 stars, but 58.9% of late orders were reviewed *before* arrival |
| Review gap after arrival | **0.46 stars** (CI 0.41 to 0.51) |
| Padding the promise 5 days | late rate 9.2% to 5.5%; net about +16,000 BRL a year under stated assumptions; breaks even at 0.51% conversion loss per added day |
| Forecast backtest | A naive benchmark (7.6% error) beat ETS (28.0%) and ARIMA (28.2%); reported as found |

Recommendation: pilot the buffer in the Northeast and North as an A/B test, message customers
before the promised date passes, and open a lost-parcel process. Full memo: [`docs/decision_memo.md`](docs/decision_memo.md).

![Review score by when the review was written](docs/dashboard/r_review_timing.png)

## The toolchain, mapped to the deliverables

| Tool | What it does here | Where |
|---|---|---|
| **SQL** (DuckDB) | Medallion build, quarantine, certified metrics, reconciliation | `sql/`, `src/pipeline.py` |
| **R** | Exact binomial CIs, Welch and Wilcoxon tests, robust (HC3) regression, logistic odds ratios, Kaplan-Meier with censoring, ETS/ARIMA backtest | `r/01_delivery_review_analysis.R`, `data/gold/r_results.json` |
| **Excel** | Decision workbook: scenario by region, break-even, A/B sample-size design; 166 live formulas, 0 errors | `excel/ParcelPulse_Decision_Workbook.xlsx` |
| **Tableau** | Real `.hyper` extracts (99,441 orders; 558 state-months) and a full dashboard spec with tested calculated fields | `tableau/` |
| **Omni** | Semantic model: certified measures defined once | `omni/` |

## Incidents

1. **Revenue fan-out.** Joining items, payments and reviews repeated each item price once per
   payment row per review row. Fixed by summing at item grain and collapsing child tables first.
2. **Survivorship in the late rate.** The delivered-only metric could not see 1,106 shipped
   parcels that never arrived. Fixed by treating them as censored-late against the data snapshot.
3. **Review timing.** 58.9% of late orders' reviews were written before arrival, so the headline
   satisfaction gap measured a missed date, not the delivery. Fixed by splitting by timing.

No rate-limit incident: the source is static files, not an API.

## Architecture

```
data/bronze (real CSV) -> BRONZE -> SILVER (types, quarantine) -> GOLD (order grain, marts)
                                                                     |
                                    R analysis, Excel workbook, Tableau extracts, Omni model
reconciliation: layer counts, revenue vs independent pandas control, late rate vs module
```

Details: [`docs/architecture.md`](docs/architecture.md), [`docs/metric_dictionary.md`](docs/metric_dictionary.md),
[`docs/cost-tradeoffs.md`](docs/cost-tradeoffs.md).

## Run it

```bash
pip install -r requirements.txt
sudo apt-get install r-base-core r-cran-tidyverse r-cran-broom r-cran-survival \
  r-cran-forecast r-cran-jsonlite r-cran-sandwich r-cran-lmtest libreoffice-calc

python -m src.pipeline                       # bronze -> silver -> gold, reconciliation, exports (writes data/gold/fct_order.csv.gz, git-ignored)
Rscript r/01_delivery_review_analysis.R      # statistics and charts
python -m src.build_hyper                    # Tableau extracts (tableau/ParcelPulse_Orders.hyper, 8 MB, git-ignored)
python -m src.build_excel                    # decision workbook (recalculated in LibreOffice)
python -m pytest tests -v                    # 40 tests
```

## Disclosed limits

* **Not opened in their own tools:** no Tableau Desktop or Omni instance was reachable. The `.hyper`
  extracts are real and read back with the Hyper API; the `.twbx` workbook was not generated; the
  Omni model is written to Omni's documented syntax and validated structurally only.
* **Observational data.** Satisfaction effects are associations. The Excel scenario assumes reviews
  respond to the date shown; the A/B test design is how to check it.
* **Three business assumptions** (value of a satisfied customer, margin, conversion loss) are
  yellow cells on the workbook's `Inputs` sheet and are not in the data.
* The dataset is a 2016 to 2018 marketplace.

## Repository

```
data/bronze/    real Olist files + SOURCE.md        sql/       numbered SQL layers
data/gold/      marts, R results (fct_order.csv.gz is generated)   src/   pipeline, metric modules, builders
r/              R analysis                          excel/     decision workbook
tableau/        .hyper extracts + dashboard spec    omni/      semantic model
docs/           incidents, memo, dictionary, charts, evidence
tests/          40 tests                            AI_DEV_LOG.md, CODING_STANDARDS.md
ParcelPulse_Executive_Summary.pptx
```

## Git history (single author)

The 13th commit is the one that adds this README and the rest of the documentation.

```
9d6299e  2026-10-12  Christian Kouadio Kouassi  Add real Olist e-commerce data and source documentation
32fe09a  2026-10-12  Christian Kouadio Kouassi  Add bronze and silver SQL layers with explicit order quarantine and tests
e0ed4bf  2026-10-12  Christian Kouadio Kouassi  Reproduce Incident 1: naive multi-join overstates item revenue by 5.01 percent
2938830  2026-10-12  Christian Kouadio Kouassi  Fix Incident 1: sum revenue at item grain, collapse child tables first, add reconciliation control
5fe130a  2026-10-12  Christian Kouadio Kouassi  Reproduce Incident 2: late rate silently drops 1,107 shipped orders that never arrived
ad95f38  2026-10-12  Christian Kouadio Kouassi  Fix Incident 2: score stuck parcels as censored-late against the data snapshot and exclude quarantined clocks
f0a7b2c  2026-10-12  Christian Kouadio Kouassi  Add gold layer, metric dictionary, reconciliation checks, and exports
cf20f27  2026-10-12  Christian Kouadio Kouassi  Add R analysis: robust regression, Kaplan-Meier with censoring, forecast backtest, charts
1d96e29  2026-10-12  Christian Kouadio Kouassi  Add Excel decision workbook: buffer scenario, break-even, A/B test design, live formulas
af2a094  2026-10-12  Christian Kouadio Kouassi  Add Tableau .hyper extracts with dashboard spec and Omni semantic model, with tests
13c3a47  2026-10-12  Christian Kouadio Kouassi  Reproduce Incident 3: 58.9 percent of late-order reviews were written before arrival, inflating the delivery effect
479c228  2026-10-12  Christian Kouadio Kouassi  Fix Incident 3: split satisfaction by review timing; post-arrival gap is 0.46 stars, not 1.74
```
# parcelpulse
