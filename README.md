# Microsoft Fabric Analytics Pipeline Demo (Local Equivalent)

A three-notebook, medallion-architecture pipeline that mirrors a typical
**Microsoft Fabric** end-to-end project, built with **open-source equivalents**
so the whole thing runs on your laptop.

> **Honesty note:** Fabric is a cloud-only SaaS platform. This repo does **not**
> call Fabric APIs. It is a local, runnable reference implementation of the
> *same concepts and patterns* an analyst uses inside Fabric, so that recruiters
> and reviewers can inspect working code end-to-end.

## Fabric <-> This Repo Mapping

| Microsoft Fabric component | Local equivalent in this repo |
|---|---|
| OneLake + Lakehouse (Delta tables) | `deltalake-python` writing to `data/bronze`, `data/silver`, `data/gold` |
| Dataflow Gen2 / Pipeline ingestion | `notebooks/01_lakehouse_ingestion.ipynb` reads CSV -> Delta bronze |
| Notebooks (Spark / Python) | These Jupyter notebooks (Polars instead of Spark) |
| Silver layer + SCD Type 2 dim | `notebooks/02_silver_transform.ipynb` — dedupe, conform, SCD2 |
| Warehouse / Semantic Model (Gold) | `notebooks/03_gold_reporting.ipynb` — star-schema aggregates |
| Power BI Direct Lake | `data/gold/` Delta tables — can be loaded by any BI tool |

## The Medallion pipeline

```
data/raw/*.csv
    |
    v
[01] Ingestion  -->  data/bronze/*  (Delta, raw + load metadata)
    |
    v
[02] Silver     -->  data/silver/*  (dedupe, typed, SCD2 customer dim)
    |
    v
[03] Gold       -->  data/gold/*    (fact + KPI tables for BI)
```

## Run it

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute notebooks/01_lakehouse_ingestion.ipynb --inplace
jupyter nbconvert --to notebook --execute notebooks/02_silver_transform.ipynb --inplace
jupyter nbconvert --to notebook --execute notebooks/03_gold_reporting.ipynb --inplace
```

Or open the notebooks interactively:

```bash
jupyter lab notebooks/
```

## What each notebook does

**`01_lakehouse_ingestion.ipynb` — Bronze**
- Reads 4 raw CSVs (sales, two customer snapshots, products)
- Writes them to Delta Lake at `data/bronze/*` with ingestion metadata
  (source file, load timestamp) — equivalent to a Fabric Dataflow Gen2
  landing files into a Lakehouse's `Tables/` section.

**`02_silver_transform.ipynb` — Silver (dimensional modelling + SCD2)**
- Deduplicates sales
- Conforms types
- Builds `dim_customer` as **Slowly Changing Dimension Type 2** by
  merging the Jan and Apr customer snapshots — tracking historical
  city / loyalty_tier changes with `valid_from`, `valid_to`, `is_current`.
- Writes conformed Delta tables at `data/silver/`.

**`03_gold_reporting.ipynb` — Gold (BI-ready)**
- Joins the star schema
- Produces `fact_sales`, `kpi_daily_revenue`, `kpi_top_products`,
  `kpi_loyalty_tier_revenue` — the shape a Power BI semantic model
  would consume via Direct Lake.
- Shows a couple of charts inline to preview what a Power BI report
  would render.

## Why Polars / deltalake-python instead of Spark?

Fabric notebooks typically run PySpark. On a laptop, PySpark is heavy and
needs a JVM. Polars + `deltalake` give the same Delta-Lake file format, the
same lazy columnar compute model, same SQL-ish API — with zero-config
startup. The **artifacts are wire-compatible Delta tables** readable by
Spark, Fabric, Databricks, or DuckDB.

## Author

Chetan Sarda — [github.com/ChetanSarda99](https://github.com/ChetanSarda99)
