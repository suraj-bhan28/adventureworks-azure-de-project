# Architecture Design Decisions
![alt text](awde_workflow-1.png)
## 1. Why Self-Hosted Integration Runtime (SHIR)
AdventureWorks (on-prem) sits inside a private network with no public inbound access. 
Azure-managed IR cannot reach into this network. SHIR is a lightweight agent installed 
inside the private network that initiates an outbound-only connection to ADF, avoiding 
the security risk of opening inbound firewall ports on the SQL Server.

## 2. Why Azure SQL Database is treated as a separate source
`sqldb-awde-source-dev` holds an independent subset of Purchasing data (PurchaseOrderHeader/
Detail), simulating a separate operational system (e.g., a cloud-hosted procurement system) 
rather than a mirrored copy of on-prem. It connects to ADF via the default Azure IR — no 
SHIR needed — demonstrating that the same metadata-driven framework works across different 
connectivity models.

## 3. Why Frankfurter REST API
Free, no authentication required, supports historical date-based queries and currency 
filtering — directly satisfying "date-based ingestion" and "parameterized API calls" 
requirements. Used to enrich AdventureWorks sales data with real FX rates for 
SalesAmountUSD/SalesAmountLocalCurrency calculations (Part 12).

## 4. Why a separate Control Database (sqldb-awde-control-dev)
Deliberately isolated from operational source data. Holds:
- `control.IngestionConfig` — generic, source-agnostic metadata (what/how to extract)
- `control.SqlSourceConfig` / `control.RestSourceConfig` — source-type-specific parameters
  (normalized sub-tables rather than one flat config table, so new source types can be 
  onboarded without altering existing tables)
- `control.IngestionState` — runtime watermark state, separated from config because it 
  changes every run and should never risk corrupting curated config definitions
- `audit.PipelineRun` — full execution audit log

This separation means ADF's Lookup Activity drives all pipeline behavior dynamically, 
with zero hardcoded per-table logic.

## 5. Why ADLS Gen2 with separate "landing" and "lakehouse" containers
- `landing/` — immutable, source-shaped raw Parquet, written directly by ADF. Acts as a 
  replayable archive; Bronze can always be rebuilt from here without re-hitting source systems.
- `lakehouse/` (bronze/silver/gold subfolders) — Databricks-managed Delta tables, 
  transformed and reshaped. Kept separate from landing because its structure is dictated by 
  Databricks processing logic, not by source system shape.
- `quarantine/` — isolated container for rejected/invalid records with rejection reasons, 
  so bad data never silently enters Silver/Gold.

## 6. Why Delta Lake / Databricks
Chosen over plain Parquet-on-Spark because we need:
- ACID transactions and `MERGE` for idempotent incremental Silver/Gold processing and SCD Type 2
- Schema evolution to handle the schema-change scenario (Part 21) without breaking pipelines
- Time travel for debugging reprocessing/rerun scenarios (Part 20)

## 7. Why Bronze / Silver / Gold medallion separation
- Bronze: raw-shaped + ingestion metadata (source_system, batch_id, ingestion_timestamp) 
  for full traceability
- Silver: cleaned, deduplicated, conformed, business-key validated — single source of truth
- Gold: dimensional star schema optimized for BI consumption, fully decoupled from source shapes

## 8. Why ExecutionGroup-based sequencing
`control.IngestionConfig.ExecutionGroup` separates reference/dimension-source tables 
(Group 1: Customer, Product, ProductCategory, etc.) from transactional/fact-source tables 
(Group 2: SalesOrderHeader/Detail, PurchaseOrderHeader/Detail). ADF processes Group 1 to 
completion before starting Group 2, ensuring dimension data exists before fact validation 
occurs in Silver.

## 9. Why optimistic concurrency on watermark updates
`control.IngestionState.StateVersion` is checked before any watermark update is committed. 
This prevents two concurrent pipeline runs from corrupting each other's watermark state — 
a stronger guarantee than a simple "update after success" rule alone.