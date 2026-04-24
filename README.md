# Microsoft Fabric Analytics Pipeline Demo (Local Equivalent)

A three-notebook medallion-architecture pipeline — bronze → silver → gold — that mirrors a Microsoft Fabric end-to-end project, built with open-source equivalents so the whole thing runs on your laptop with `pip install -r requirements.txt`.

> **Honest framing up front:** Fabric is a cloud-only SaaS platform priced by capacity. This repo does **not** call Fabric APIs. It is a local, runnable reference implementation of the *same architectural patterns* an analyst would use inside Fabric, so reviewers can inspect working code without me buying a Fabric capacity SKU.

## Why I built this

Microsoft Fabric is the all-in-one analytics platform — OneLake + Lakehouse + Dataflow Gen2 + Warehouse + Power BI, stitched together, billed by capacity unit. Fabric is on a lot of Canadian public-sector and enterprise job descriptions right now. I wanted to demonstrate Fabric competency without paying for capacity, so I built a local equivalent that exercises the same architectural pattern: raw files land in a lakehouse, get cleaned and conformed into a silver layer with dimensional modelling (including a real Slowly Changing Dimension Type 2), and get rolled up into a gold star-schema that a BI tool would consume.

This is the pattern. The scale differs. The *ideas* transfer 1:1 from this repo to Fabric, Databricks, or any Spark-on-Lakehouse stack. The *scale* does not — you can't run a 50-terabyte enterprise workload on a MacBook and I'm not pretending otherwise.

## Why I used Polars + Delta Lake instead of Spark

Fabric notebooks typically run PySpark. On a laptop, PySpark is heavy — it needs a JVM, a Spark session, a startup tax measured in seconds per notebook. Polars + the `deltalake` Python package give you:

- The same Delta Lake file format — wire-compatible Delta tables that Spark, Fabric, Databricks, or DuckDB can read.
- The same lazy columnar compute model.
- A SQL-ish DataFrame API that ports cleanly to PySpark.
- Zero-config startup.

The artifacts in `data/bronze/`, `data/silver/`, `data/gold/` are real Delta tables. You could point a Fabric Lakehouse at them and it would read them. The engine differs; the format doesn't.

## The medallion architecture — what each layer does

```
data/raw/*.csv
    |
    v
[01] Bronze   -->  data/bronze/*   (Delta, raw + ingestion metadata)
    |
    v
[02] Silver   -->  data/silver/*   (dedupe, conform types, SCD2 customer dim)
    |
    v
[03] Gold     -->  data/gold/*     (fact + KPI tables, BI-ready)
```

**Bronze.** Source-of-truth raw data, but in Delta format with ingestion metadata (`_ingest_ts`, `_source_file`) attached. I never transform bronze — if something goes wrong downstream, I can always rebuild silver and gold from bronze without re-running the source extraction. Fabric equivalent: a Lakehouse `Tables/` section populated by Dataflow Gen2.

**Silver.** Deduped, conformed, type-safe. This is where business rules live. The centerpiece is `dim_customer`, built as **Slowly Changing Dimension Type 2** by merging two customer snapshots (Jan and Apr). When a customer changes city or loyalty tier, silver records the old value with a `valid_to` and a new row with `valid_from` and `is_current = true`. Fabric equivalent: silver-layer Dataflow Gen2 transforms and dimensional modelling.

**Gold.** Star schema and KPI aggregates, shaped for a BI tool. `fact_sales` joined to `dim_customer` / `dim_product` / `dim_date`, plus pre-aggregated KPIs: `kpi_daily_revenue`, `kpi_top_products`, `kpi_loyalty_tier_revenue`. Fabric equivalent: Fabric Warehouse + Power BI Direct Lake semantic model.

## Fabric component mapping

| Microsoft Fabric component | Local equivalent in this repo |
|---|---|
| OneLake + Lakehouse (Delta tables) | `deltalake-python` writing to `data/bronze`, `data/silver`, `data/gold` |
| Dataflow Gen2 / Pipeline ingestion | `notebooks/01_lakehouse_ingestion.ipynb` reads CSV → Delta bronze |
| Notebooks (Spark / Python) | These Jupyter notebooks (Polars instead of Spark) |
| Silver layer + SCD Type 2 dim | `notebooks/02_silver_transform.ipynb` — dedupe, conform, SCD2 |
| Warehouse / Semantic Model (Gold) | `notebooks/03_gold_reporting.ipynb` — star-schema aggregates |
| Power BI Direct Lake | `data/gold/` Delta tables — can be loaded by any BI tool |
| Matplotlib inline charts | Stand-in for a Power BI report visual |

## Notebook 1 — Lakehouse ingestion (bronze)

Reads four raw CSVs (sales, two customer snapshots, products) with Polars and writes them to Delta Lake at `data/bronze/*` with ingestion metadata attached: `_ingest_ts` (load timestamp) and `_source_file` (provenance). This is the lineage bread-crumb trail — when somebody asks "where did this number come from", you can trace it back to the exact file and load timestamp.

Why Polars over pandas here: Polars is faster, supports lazy evaluation (so the query planner can push filters and projections down before materializing), and has native Delta Lake write support. On a 150-row demo it doesn't matter; the habit matters.

## Notebook 2 — Silver transform (the SCD2 notebook)

Silver is where the real work happens. Three jobs:

1. **Dedupe `sales`** — the raw CSV has a few exact-duplicate rows; drop them.
2. **Null-fill** — null product categories become `"unknown"` rather than propagating nulls through downstream joins.
3. **Build `dim_customer` as SCD Type 2** — the centerpiece.

**SCD Type 2 in plain English:** when a customer's attribute changes (say, their `loyalty_tier` goes from Silver to Gold), you don't overwrite the old value — you close out the old row with a `valid_to` timestamp and `is_current = false`, and insert a new row with `valid_from = today`, `valid_to = null`, `is_current = true`. Now every historical fact can be joined back to the correct dimension state *at the time the fact happened*. Without SCD2, a sales report from January would suddenly start showing every January order under "Gold tier" because that's the customer's current tier today. With SCD2, January orders stay joined to the January tier. It's bookkeeping — not hard, just meticulous.

The merge logic:

```python
# pseudo-code of the merge
changed = (
    april.join(jan, on="customer_id")
         .filter(pl.col("city_april") != pl.col("city_jan")
              | pl.col("loyalty_tier_april") != pl.col("loyalty_tier_jan"))
)

closed_rows = jan_current_rows.filter(
    pl.col("customer_id").is_in(changed["customer_id"])
).with_columns(
    valid_to = pl.lit(APRIL_DATE),
    is_current = pl.lit(False),
)

new_rows = april_rows_for_changed_customers.with_columns(
    valid_from = pl.lit(APRIL_DATE),
    valid_to = pl.lit(None),
    is_current = pl.lit(True),
)

dim_customer = pl.concat([unchanged, closed_rows, new_rows])
```

After the merge, `dim_customer` contains at most two rows per customer — the pre-change row (closed out) and the current row. Fact tables join on `customer_id` + a date range filter, which gives you correct point-in-time attribution.

## Notebook 3 — Gold reporting (star schema + KPIs)

Builds the star schema (`fact_sales` joined to `dim_customer`, `dim_product`, `dim_date`) and three KPI aggregates:

- `kpi_daily_revenue` — daily total revenue, with a matplotlib line chart inline so the notebook is self-visualizing.
- `kpi_top_products` — products ranked by revenue.
- `kpi_loyalty_tier_revenue` — revenue split by loyalty tier (point-in-time correct thanks to SCD2).

The matplotlib charts are the local stand-in for a Power BI report visual. In actual Fabric, you'd point a Power BI semantic model at the gold Delta tables using Direct Lake mode — no data duplication, no refresh, the BI layer reads Delta files directly. That's the unlock Fabric sells.

## Run it

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute notebooks/01_lakehouse_ingestion.ipynb --inplace
jupyter nbconvert --to notebook --execute notebooks/02_silver_transform.ipynb --inplace
jupyter nbconvert --to notebook --execute notebooks/03_gold_reporting.ipynb --inplace
```

Or open them interactively:

```bash
jupyter lab notebooks/
```

Inspect the Delta tables directly from Python:

```python
from deltalake import DeltaTable
import polars as pl
dt = DeltaTable("data/gold/fact_sales")
pl.from_arrow(dt.to_pyarrow_table()).head(10)
```

## What I learned

- **Medallion architecture is default-right for most analytics pipelines.** Each layer has one job (raw / conformed / BI-ready), and that separation is what makes the pipeline maintainable. I'll reach for it even on small projects now.
- **SCD2 via snapshot-merge is less scary than it looks once you accept it's just bookkeeping.** Close the old row, open a new one, track `valid_from` / `valid_to` / `is_current`. That's it. The scary part is disciplining yourself to do it every time an attribute changes.
- **Polars + Delta is a legit Spark-free stack for small-to-medium data.** Zero startup tax, wire-compatible output, SQL-ish API. For anything that fits in memory, I'd reach for this before PySpark.
- **Direct Lake mode (in actual Fabric) is the unlock for Power BI** — no semantic-model duplication, the BI engine reads Delta directly. That's genuinely novel and a big reason Fabric is getting traction.
- **I'd never build a production data platform this way, but the mental model transfers cleanly** to Spark + Databricks or Spark + Fabric. The point of this repo isn't "use this in prod" — it's "prove I understand the pattern and can operate inside a Fabric shop."

## Honest limits

Running this locally proves I understand the architecture. Running it on Fabric would add cloud scalability, capacity-based pricing, auto-optimization, native BI integration, governance via Purview, workspace-level permissions, and all the operational wiring that actual enterprise analytics platforms need. The *ideas* transfer 1:1; the *scale* does not. This is a portfolio piece, not a production pattern.

## What's next

- Swap Polars for DuckDB in a branch and benchmark the silver/gold builds — both are Delta-native, curious which is faster.
- Deploy to actual Fabric when I can get capacity access through a work account or a free trial.
- Add dbt on top of the silver/gold layer so transforms are version-controlled and testable — Fabric supports dbt, so this would port directly.
- Add great-expectations or dbt-style tests on the silver layer for null-count and referential-integrity guards.

## Author

Chetan Sarda — [github.com/ChetanSarda99](https://github.com/ChetanSarda99)
