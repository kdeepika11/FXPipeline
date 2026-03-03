**Asset Lakehouse - FX Rates**

This project implements a thin-slice lakehouse pipeline using free/public FX rates (ECB-based via Frankfurter) to demonstrate an enterprise-style architecture:

- One source → one Gold table

- Parameterized ingestion (ADF) into ADLS Gen2 Bronze

- Transformations in Databricks into Silver (conformed) and Gold (query-ready)

- Data quality gates + freshness checks (SLO/error budget)

- Governed access using Unity Catalog (External Locations + credentials)

**Setup: Azure Databricks + Unity Catalog + ADLS Gen2 (Working Steps)**

**Prerequisites**

- Azure Databricks workspace (Trial/Premium is fine) in East US

- ADLS Gen2 storage account: stassetlakehouse (Hierarchical namespace enabled)

- ADLS containers created:

- bronze, silver, gold (optional: meta, logs)

- Unity Catalog enabled in Databricks (Catalog Explorer visible)

**1) Create Databricks compute (cost-safe)**

- Databricks → Compute → Create compute

- Recommended settings:

  - Single node: ✅ On

  - Auto-terminate: ✅ 10 minutes

  - Runtime: ✅ LTS

  - Smallest available node type

**2) Use Unity Catalog’s Managed Identity (Access Connector)**

- This workspace is configured to use Azure Managed Identity for Unity Catalog (Service Principal option not available in credential types).

- Databricks identity used:

  - Managed identity / access connector: unity-catalog-access-connector (name may vary)

**3) Assign Azure IAM roles to the Managed Identity**

- Azure Portal → Storage account stassetlakehouse → Access Control (IAM) → Add role assignment
- Assign the following roles to the managed identity (unity-catalog-access-connector):

  **Role	Why it’s needed**
    - Storage Blob Data Contributor	Read/write/list/delete files in ADLS containers
    - Storage Queue Data Contributor	Required for “Managed File Events” validation
    - EventGrid EventSubscription Contributor	Required for “Managed File Events” validation

**Note:** these extra roles were required because Databricks validates file events when creating External Locations in this workspace.

**4) Set ADLS Gen2 ACLs (POSIX permissions)**

- Even with IAM roles, ADLS Gen2 may block access unless ACLs are set.

- For each container (bronze, silver, gold):

- Azure Portal → Storage account → Containers → select container

- Click Manage ACL

- Add principal: unity-catalog-access-connector

- Grant permissions:

    - Read (r) ✅

    - Write (w) ✅

    - Execute (x) ✅ (required to traverse/list folders)

- Apply in both tabs:

    - Access permissions ✅

    - Default permissions ✅ (so new folders inherit ACLs)

**5) Create Unity Catalog External Locations**

Databricks → Catalog → External Data → External Locations → Create

Create these locations using the existing managed identity credential (example name: adb_assetlakehouse_dev):

  - loc_stassetlakehouse_bronze → abfss://bronze@stassetlakehouse.dfs.core.windows.net/

  - loc_stassetlakehouse_silver → abfss://silver@stassetlakehouse.dfs.core.windows.net/

  - loc_stassetlakehouse_gold → abfss://gold@stassetlakehouse.dfs.core.windows.net/

Test connection should show success for: Read / List / Write / Delete / Path Exists / HNS enabled.
If “File Events Read” shows Skipped, that’s OK for batch pipelines.

**6) Grant Databricks permissions on External Locations**

For each external location:

  - Open location → Permissions

  - Grant your user (or a group):

    - READ FILES

    - WRITE FILES

    - (Optional) CREATE EXTERNAL TABLE

**7) Validation**

Run this from a Databricks notebook attached to the cluster:

dbutils.fs.ls("abfss://bronze@stassetlakehouse.dfs.core.windows.net/")

Expected: directory listing (empty is fine).
---------------------------------------------------------------------------
**Data model**
**Bronze (raw, 1:1 source)**

Stores raw JSON response from API

**Path pattern:**

abfss://bronze@stassetlakehouse.dfs.core.windows.net/
fx_rates/base=<BASE>/run_date=<YYYY-MM-dd>/fx_rates_<BASE>_<YYYY-MM-dd>.json
**Silver (conformed entity)**

**Table: assetlakehouse_dev.silver.fx_rates**

**Columns:**

- rate_date (DATE)

- base_ccy (STRING)

- quote_ccy (STRING)

- fx_rate (DOUBLE)

- ingest_ts (TIMESTAMP)

- source_file (STRING)

- run_date (DATE)

**Gold (use-case table)**

**Table: assetlakehouse_dev.gold.fx_daily_rates

Columns:**

- rate_date, base_ccy, quote_ccy, fx_rate

- ingest_ts

- freshness_hours

- is_required_ccy

- run_id, run_ts

**Metrics / Use case (Gold)**

**Question:** “What are the latest FX rates for required currencies, and is the feed complete and fresh?”

**Gold supports:**

- daily rates per currency pair

- freshness tracking (freshness_hours)

- completeness for required currency list

- Default required symbols:
- EUR,GBP,JPY,CHF,CAD,AUD,USD

Note: if the base currency is USD, we ensure USD->USD = 1.0 in Silver.

- Freshness and quality (SLO / Error budget)
- Freshness SLO (dev)

- Pipeline should load latest available business day (API /latest)

- Freshness tracked in Gold as freshness_hours

**Data quality gates**

- DQ notebook writes results to:
- assetlakehouse_dev.meta.dq_run_results

**CRITICAL checks**

- No nulls in key fields (rate_date/base_ccy/quote_ccy/fx_rate)

- fx_rate > 0

- Required currencies present for the run date/base

**WARN checks**

- Freshness over threshold (e.g., > 24 hours)

- If any CRITICAL check fails, the workflow fails.
**ADF pipeline**
**Pipeline: pl_ingest_fx_to_bronze**

**Parameters:**

- p_run_date (String) e.g., 2026-03-03 (used for landing path/audit)

- p_base_ccy (String) default USD

- p_symbols (String) default EUR,GBP,JPY,CHF,CAD,AUD,USD

** HTTP endpoint (avoid weekend 404s by using latest):**

- Base URL (linked service): https://api.frankfurter.dev/v1/

**Relative URL constructed as:**

latest?from=<BASE>&to=<SYMBOLS>

**Copy activity:**

Source: HTTP

Sink: ADLS Gen2 bronze container

**Folder**:
fx_rates/base=<BASE>/run_date=<p_run_date>/

File:
fx_rates_<BASE>_<p_run_date>.json

**Databricks notebooks**

- 01_fx_bronze_to_silver

- Reads Bronze JSON

- Parses rates into rows (MapType schema)

- Adds metadata columns

- Writes to assetlakehouse_dev.silver.fx_rates

- Updates schema registry table assetlakehouse_dev.meta.schemas (optional)

- 02_fx_silver_to_gold

- Filters Silver for rate_date and base currency

- Writes query-ready results to assetlakehouse_dev.gold.fx_daily_rates

- 03_fx_dq_gates

- Runs DQ checks and logs results to assetlakehouse_dev.meta.dq_run_results

- Fails on CRITICAL issues

**Databricks Workflow (triggered by ADF)
Workflow job parameters (job-level)**

- run_date

- base_ccy

- symbols

**Notebook widget parameters (task-level mapping)**

Each task maps job parameters into notebook widgets:

p_run_date ← {{job.parameters.run_date}}

p_base_ccy ← {{job.parameters.base_ccy}}

p_required_symbols ← {{job.parameters.symbols}}

**ADF → Databricks override**

ADF Databricks Job activity passes:

run_date = @pipeline().parameters.p_run_date

base_ccy = @pipeline().parameters.p_base_ccy

symbols = @pipeline().parameters.p_symbols

These values override Databricks defaults for each triggered run.

**How to run**
**Manual test (recommended first)**

Trigger ADF pipeline for a run date (no quotes):

p_run_date = 2026-03-03

Run Databricks notebooks in order:

01_fx_bronze_to_silver

02_fx_silver_to_gold

03_fx_dq_gates

**End-to-end automated run**

**ADF pipeline runs Copy → triggers Databricks Workflow job → writes Silver/Gold → logs DQ.**

**Validation queries**
SELECT * FROM assetlakehouse_dev.silver.fx_rates
ORDER BY ingest_ts DESC
LIMIT 20;

SELECT * FROM assetlakehouse_dev.gold.fx_daily_rates
ORDER BY run_ts DESC, quote_ccy;

SELECT * FROM assetlakehouse_dev.meta.dq_run_results
ORDER BY run_ts DESC;
