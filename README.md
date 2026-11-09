# Workforce Analytics @ 7Trents — Databricks Lakehouse Platform

**End-to-end data engineering pipeline** that turns daily workforce-management files (work, shifts, and shrinkage) into a governed, validated, Power BI–ready data model using the **medallion architecture** (Bronze → Silver → Gold) on **Databricks, Delta Lake, and Unity Catalog**.

This was a group data-engineering project for 7Trents. I owned the **Silver layer**: data cleaning, dimension processing, reconstruction of daily fact snapshots from full + delta loads, and a full data-quality assessment framework.

## 🎯 Business problem

Workforce managers at 7Trents need to measure **time and work utilization**: what work was planned, what actually happened, and how much capacity was lost to shrinkage. The source data arrives as daily files where:

* The **first file per fact is a full load** — the complete state of the world.
* Every **later daily file is a delta** — only what changed since the previous day.

The key analytical requirement is **plan adherence**: comparing the plan *as it was known on a given day* (e.g. Monday's snapshot shows 3 scheduled jobs) against the *actual outcome later* (Tuesday's delta shows only 2 completed). That means each day's state must be preserved historically — you can't just keep the latest row per record.

## 🏗️ Architecture

```
GitHub repo (source CSVs)
        │
        ▼  ingestion/dataIngFrmGithub.py — GitHub API, download to UC Volume
┌─────────────────────────────────────────────────────────────┐
│ BRONZE  workforce_analytics.bronze                           │
│  • Raw Delta tables (7 entities)                             │
│  • Lineage columns: _source_file, _ingested_at              │
└─────────────────────────────────────────────────────────────┘
        │
        ▼  bronze/bronze.py — recursive CSV discovery, load + inspect
┌─────────────────────────────────────────────────────────────┐
│ SILVER  workforce_analytics.silver                           │
│  • Dims: trim/cast/dedupe → clean conformed dimensions       │
│  • Facts: FULL load + daily DELTAS → complete dated snapshots│
│    keyed by record ID + snapshot_date                        │
│  • 9-step data-quality assessment                            │
└─────────────────────────────────────────────────────────────┘
        │
        ▼  gold/ — conformed star schema
┌─────────────────────────────────────────────────────────────┐
│ GOLD    workforce_analytics.gold                             │
│  • Power BI data model (facts joined to dimensions)          │
└─────────────────────────────────────────────────────────────┘
```

## 📊 Data model

| Layer | Table | Description |
| --- | --- | --- |
| Silver | `fact_work` | Jobs/work items: people, county, status (Scheduled / Confirmed / Completed), cancellation, scheduled times — 5 daily snapshots |
| Silver | `fact_shift` | Employee shifts with scheduled start/end — 5 daily snapshots |
| Silver | `fact_shrinkage` | Lost-capacity events (start/end datetime, shrinkage type) — 5 daily snapshots |
| Silver | `dim_people` | 300 people (employment type, job role) |
| Silver | `dim_county` | 10 counties (urban/rural) |
| Silver | `dim_cancellation_reason` | 10 reasons with severity, reason group, customer-driven flag |
| Silver | `dim_shrinkage_type` | 5 shrinkage categories |

**Snapshot reconstruction:** each fact table's five daily files (01–05 Apr 2026) are applied chronologically — the full load establishes the baseline and each delta updates it — producing a complete snapshot per day that grows from **7,000 records (Apr 1) to 8,120 records (Apr 5)**. Every (record ID, snapshot date) pair is unique, enabling point-in-time queries for plan-adherence reporting.

## ✅ Data-quality framework

The Silver layer includes a 9-step DQ assessment applied across all 7 tables:

1. Row-count and schema profiling per table
2. Null / empty-string / whitespace-only checks on **every column** of every table
3. Duplicate business-key checks (fact grain = record ID + snapshot_date; dim grain = natural key)
4. Business-rule validation — e.g. `scheduled_end > scheduled_start`, `end_datetime > start_datetime`
5. Snapshot distribution — all 5 snapshot dates present and populated in each fact table
6. Referential integrity — every fact FK (`people_id`, `county_id`, `cancellation_id`, `shrinkage_type_id`) matched to its dimension (left-anti join)
7. Categorical-value profiling — statuses, employment types, severity, urban/rural, categories
8. Snapshot-date vs. filename-date consistency (date parsed from `_source_file` must equal `snapshot_date`)
9. Idempotent, validated promotion of snapshot tables to the main Silver tables

**Results:** zero duplicate keys, zero missing/invalid time ranges, zero unmatched foreign keys, zero snapshot-date mismatches across all silver tables.

## 🛠️ Tech stack

* **Databricks** (notebooks, Unity Catalog, Volumes)
* **PySpark / Spark SQL** — transformations and validation
* **Delta Lake** — ACID tables, schema evolution on overwrite
* **GitHub API + `requests`** — automated source-file ingestion
* **Power BI** — downstream consumption of the Gold star schema

## 📁 Repository structure

```
workforce_analytics-7trent/
├── ingestion/
│   └── dataIngFrmGithub.py      # GitHub API → UC Volume download
├── bronze/
│   └── bronze.py                # Volume → raw Delta tables + inspection
├── silver/
│   ├── dimension/
│   │   ├── silv_dim_people.py
│   │   ├── silv_dim_county.py
│   │   ├── silv_dim_shrinkageType.py
│   │   └── silv_dim_cancellationReas.py  # + full silver DQ assessment
│   └── fact/
│       ├── new_silv_fact_work.py       # FULL + DELTA → dated snapshots
│       └── shift_shrinkage.py          # shift & shrinkage snapshots
├── gold/
│   ├── dimension/  # gold dim notebooks
│   └── fact/       # gold fact notebooks (Power BI model)
└── source/         # raw CSVs (facts + dimensions)
```

## ▶️ How to run

1. Run `ingestion/dataIngFrmGithub.py` — creates the catalog/schema/volume and downloads the CSVs from GitHub.
2. Run `bronze/bronze.py` — loads raw Delta tables with lineage metadata.
3. Run the Silver dimension notebooks, then the Silver fact notebooks — validated snapshots are promoted to the main silver tables.
4. Run the Gold notebooks to build the star schema for Power BI.

Requires a Databricks workspace with Unity Catalog enabled and `CAN USE`/write permissions on the `workforce_analytics` catalog.

