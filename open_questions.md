# Open Questions

Answers refer to `case_study.ipynb`: a three-layer Delta lake on Databricks (Unity Catalog) where all DDL,
transformations and validations are Spark SQL.

## 1. What steps are missing to industrialize the solution?

- **Incremental ingestion:** Auto Loader or `COPY INTO` instead of a full reload, plus audit columns
  (`_source_file`, `_ingested_at`) on raw tables.
- **Idempotent incremental writes:** `MERGE` on `event_id` / `item_id`, or `replaceWhere` per `event_year`, instead
  of `INSERT OVERWRITE`; recompute only the affected mart years for late-arriving events.
- **History:** SCD2 on `item` if price or name can change, so historical marts stay correct.
- **Data quality as a gate:** the notebook's checks become blocking expectations (Lakeflow Declarative Pipelines,
  dbt tests or Great Expectations), with a quarantine table for unparseable rows and unknown parameters.
- **Orchestration:** a Databricks Workflow `raw → curated → mart` with retries, alerting and freshness SLAs.
- **Environments and CI/CD:** parameterised `dev` / `test` / `prod` catalogs, code in Git, deployment via
  Databricks Asset Bundles from GitHub Actions or Azure DevOps.
- **Infrastructure as code:** Terraform for catalogs, schemas, grants, warehouses and jobs.
- **Governance:** Unity Catalog grants per layer (analysts read `mart` only), service principals for jobs,
  masking of `user_id` where not needed, retention policy.
- **Table maintenance:** `OPTIMIZE` / `VACUUM` or predictive optimisation; liquid clustering on
  `event_date, item_id` once volumes grow.
- **Cost control:** job compute or serverless instead of all-purpose clusters, auto-termination, cluster policies,
  cost tags per pipeline, and incremental processing so each run pays only for new data.

## 2. If the solution were implemented in dbt-core, how would the architecture change? Would other cloud resources be needed?

Because the transformations are already Spark SQL, the move to dbt is mostly a **re-packaging**, not a rewrite.

| Notebook today | dbt-core |
|---|---|
| `CREATE TABLE` DDL + `INSERT OVERWRITE … SELECT` | One model per table: only the `SELECT`. dbt generates the DDL/DML from the model `config()` (`materialized`, `partition_by`, `file_format='delta'`) |
| Hard-coded table names, cell order defines dependencies | `{{ source() }}` / `{{ ref() }}`; dbt builds the DAG and runs it in order |
| Column comments in DDL | `schema.yml` descriptions, published with `dbt docs` |
| Validation cell (10 SQL checks) | dbt tests: `unique`, `not_null`, `relationships`, plus singular tests for reconciliation and rank |
| Download + CSV read + `raw` load | **Stays outside dbt.** dbt only transforms data already in the warehouse; raw tables are loaded by a Databricks job (Auto Loader / `COPY INTO`) and declared as dbt `sources` |

Project layout:

```
models/
  sources.yml          raw.item, raw.event
  staging/             stg_item.sql, stg_event.sql      (layer 2)
  marts/               top_item.sql                     (layer 3)
  schema.yml           descriptions + tests
tests/                 mart_reconciles_with_fact.sql, rank_one_is_max.sql
```

**Additional resources:**
- A **Databricks SQL warehouse** (serverless) as the compute dbt connects to through the `dbt-databricks` adapter.
- A **runner** for `dbt build`: simplest is a Databricks Workflow *dbt task*; alternatively a CI container
  (GitHub Actions / Azure DevOps) or Airflow.
- A **service principal** with its secret in a secret store, and a **Git repo with CI** (`dbt build` on PR).
- Optional: storage for `manifest.json` (slim CI) and hosting for `dbt docs`.

No new data storage; tables stay in Delta in the same Unity Catalog.

## 3. What would dbt-core bring? Upsides and downsides

**Upsides**
- Tests defined next to the models and run on every build, replacing the hand-written validation cell.
- Environments (dev / CI / prod) from the same code via targets.
- Incremental models (`merge`) and snapshots (SCD2) with little code, which covers most of question 1.
- Plain-SQL diffs make code review easy; analysts can contribute.

**Downsides**
- No ingestion: layer 1 still needs a separate job, so there are two tools to operate.
- dbt-core has no scheduler, UI or hosted docs; these must be built (or bought with dbt Cloud).
- Jinja and macros can make simple SQL harder to read and debug.
- For three models the gain is modest; it pays off as the number of models and contributors grows.

## 4. Effort estimate for a dbt-core implementation

Assumes an existing Databricks workspace and Unity Catalog, and reuses the existing SQL.

| Work item | Person-days |
|---|---|
| dbt project setup, adapter, dev/prod targets, service principal, SQL warehouse | 0.5 |
| Port the 3 models (`stg_item`, `stg_event`, `top_item`) and declare sources | 0.5 |
| Tests and documentation (port the 10 validation checks) | 0.5 |
| Raw ingestion job (Auto Loader / `COPY INTO`) | 0.5 |
| CI/CD and orchestration (Workflow with ingestion + dbt task, alerting) | 1.0 |
| Validation against the notebook output, review | 0.5 |
| **Total** | **≈ 3.5 person-days** |

The range is 2.5–5 days depending on whether CI/CD and platform standards already exist. A manual-run PoC
(models and tests only) takes about 1 day, because the SQL is already written.
