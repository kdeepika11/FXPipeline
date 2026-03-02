# FXPipeline
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
