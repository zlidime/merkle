# Data Engineer Case Study: Item Views Data Lake

A three-layer Delta data lake on Databricks built from `item.csv` and `event.csv`, ending in the `top_item` datamart.

## Contents

| File | Description |
|---|---|
| `case_study.ipynb` | The full solution: ingestion, DDLs, all three layers, the datamart, validation checks. Every step is documented in Markdown. |
| `open_questions.md` | Answers to the open questions (industrialisation, dbt-core architecture, pros/cons, effort estimate). |

## Data lake layout (Unity Catalog `workspace`)

| Layer | Table | Grain | Notes |
|---|---|---|---|
| 1 `raw` | `raw.item` | 1 row per source line | All `STRING`, original column names |
| 1 `raw` | `raw.event` | 1 row per source line | All `STRING`, original column names (incl. `event.payload`) |
| 2 `curated` | `curated.item` | 1 row per `item_id` | Typed dimension, empty strings → `NULL`, `DECIMAL` price |
| 2 `curated` | `curated.event` | 1 row per `event_id` | JSON payload flattened, parameters pivoted into typed columns, **partitioned by `event_year`** |
| 3 `mart` | `mart.top_item` | 1 row per `item_id` × `view_year` | Total views, `RANK()` within year, most used platform |

## How to run

1. Sign up for **Databricks Free Edition** and import `case_study.ipynb` (Workspace → Import).
2. Attach it to serverless compute and choose **Run all**.
3. The notebook downloads both files from the public S3 bucket into a Unity Catalog volume. If outbound HTTP is
   blocked in your workspace, upload the two CSVs to `workspace.raw.landing` through the Catalog UI and skip the
   download cell.

The notebook ends with assertions (row-count reconciliation, key uniqueness, mart grain, views reconciliation,
rank sanity). A successful run prints `All validation checks passed.`

## Key assumptions

- Each event is spread over several source rows (one per parameter); layer 2 pivots them into one row per event.
- A *view* is an event with `event_name = 'view_item'`; the viewed item is its `item_id` parameter.
- The mart covers every item × every year with view events; items without views in a year show `0` views and `NULL` platform.
- Rank uses `RANK()` (ties share a rank). Platform ties are broken alphabetically for a deterministic result.
- Exact duplicate event rows are removed; unparseable payloads, unmapped parameters and orphan item references are reported, not hidden.
