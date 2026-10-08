# Lakeflow Connect + DAB + GitHub Actions Ingestion Framework (POC)

This repository documents a simple, end-to-end ingestion framework for a POC using:
- Lakeflow Connect for source ingestion
- Databricks Asset Bundles (DAB) for deployment
- GitHub Actions for CI/CD automation
- Databricks workspace for orchestration and storage

The goal is to show how data from multiple sources such as SQL Server, Oracle, flat files, and cloud object storage can be moved into a Databricks Lakehouse architecture in a repeatable and governable way.

## 1. High-level architecture

```mermaid
flowchart LR
    A[GitHub Repository] --> B[GitHub Actions]
    B --> C[Databricks Asset Bundle (DAB)]
    C --> D[Databricks Workspace]
    D --> E[Lakeflow Connect / Ingestion Jobs]
    E --> F[Source Systems]
    F --> G[Extract & Validate]
    G --> H[Raw / Bronze Layer]
    H --> I[Transform / Silver Layer]
    I --> J[Curated / Gold Layer]
    J --> K[Unity Catalog / BI / Apps]

    F1[SQL Server] --> E
    F2[Oracle] --> E
    F3[Files / CSV / JSON / Parquet] --> E
    F4[Cloud Storage] --> E

    subgraph Security
        S1[Secrets / Token / Key Vault]
        S2[RBAC / Permissions]
    end

    B --> S1
    D --> S2
    E --> S1
```

## 2. End-to-end flow explained

### Step 1: Source code is stored in GitHub
All ingestion logic, configuration files, jobs, and deployment artifacts live in a GitHub repo.

This includes:
- connector configuration
- job definitions
- transformation logic
- deployment descriptors
- environment variables and secrets references

### Step 2: GitHub Actions runs the CI/CD workflow
GitHub Actions acts as the automation layer.

Typical actions:
- validate repository code
- run unit tests / lint checks
- package deployment artifacts
- authenticate to Databricks
- trigger DAB deployment

This ensures that code changes are tested and deployed in a controlled way before reaching production.

### Step 3: Databricks Asset Bundles (DAB) deploy the jobs
DAB is used to package and deploy Databricks resources such as:
- jobs
- scripts
- notebooks
- clusters or job settings
- pipeline definitions

The bundle makes the deployment repeatable across environments such as dev, test, and prod.

### Step 4: Databricks workspace receives the deployed bundle
The Databricks workspace becomes the operational control plane.

At this stage, the workspace knows:
- which jobs exist
- what the ingestion pipeline does
- what source systems to connect to
- where data should land

### Step 5: Lakeflow Connect reads from source systems
Lakeflow Connect is the ingestion engine that connects to external systems.

Examples of supported sources in a POC context:
- Microsoft SQL Server
- Oracle Database
- CSV / JSON / Parquet files
- Azure Blob / ADLS
- S3 / cloud storage
- SaaS sources when supported

The connector handles source system connectivity, extraction, metadata understanding, and staging logic.

### Step 6: Data is extracted and validated
Once connected, the ingestion layer performs:
- authentication and authorization
- schema inspection
- row extraction
- column mapping
- basic validation for nulls, duplicates, and format issues
- checkpointing or watermark tracking for incremental loads

This stage is critical because data quality issues at ingestion are much easier to handle before they reach downstream processing.

### Step 7: Data lands in a raw staging area
The extracted data is written into a raw or bronze layer in the lakehouse.

Typical landing locations:
- ADLS / S3 / DBFS paths
- Delta tables in a raw schema
- Unity Catalog-managed storage

This layer preserves the original source data as-is or near-is.

### Step 8: Transformations create the curated data model
From raw data, downstream jobs apply:
- schema normalization
- cleansing
- deduplication
- business rules
- joins with reference data
- incremental processing

This results in silver and gold layers depending on the architecture.

### Step 9: Data is published to analytics consumption layers
Final curated datasets are made available to:
- BI tools
- dashboards
- downstream data products
- machine learning workflows
- enterprise analytics applications

These datasets are typically stored as Delta tables and governed through Unity Catalog.

## 3. Practical source flow examples

### A. SQL Server ingestion flow
1. GitHub repo defines SQL Server connection details
2. GitHub Actions deploys bundle to Databricks
3. DAB deploys SQL ingestion job
4. Lakeflow Connect reads SQL Server tables
5. Data is validated and written to raw Delta tables
6. Cleansing and transformations happen in later jobs

### B. Oracle ingestion flow
1. Oracle JDBC connection details are stored in secrets
2. DAB deploys Oracle ingestion job
3. Connector reads source tables incrementally or full-load based on configuration
4. Data lands in bronze storage
5. Standardized, validated data is pushed to curated tables

### C. File ingestion flow
1. Files are placed in a landing path or object storage
2. Ingestion job reads CSV / JSON / Parquet
3. File metadata and row validation are applied
4. Data is staged in a raw folder
5. Transformation and publishing follow the same pattern

## 4. Security and governance model

A production-ready framework should include:
- Databricks secret scopes for source credentials
- GitHub repository secrets for Databricks auth
- RBAC / role-based access controls
- Unity Catalog governance for table permissions
- audit logs and monitoring
- environment separation for dev / test / prod

## 5. How the POC fits together

For a POC, the simplest pattern is:
- one GitHub repo
- one DAB deployment package
- multiple ingestion jobs per source type
- common helper functions for logging, validation, and metadata tracking
- one target landing zone in Databricks

This gives a clean architecture that can scale as more source systems are added.

## 6. Suggested POC scope

A manageable POC can include:
- SQL Server source
- Oracle source
- Flat file source
- one bronze landing zone
- one silver transformation layer
- one GitHub Actions workflow
- DAB deployment for all jobs

## 7. Summary

This architecture creates a clean, repeatable ingestion lifecycle:

GitHub repo -> GitHub Actions -> DAB deploy -> Databricks jobs -> Lakeflow Connect -> Source systems -> Raw/bronze -> Transformations -> Curated outputs

This is the core of a modern Lakehouse ingestion framework that can grow from a small POC into a scalable enterprise pattern.

## 8. Recommended next steps

1. Add source-specific connector configs
2. Define bronze / silver / gold schemas
3. Create DAB job definitions
4. Add GitHub Actions workflow for deployment
5. Validate end-to-end execution for each source

---

This repository is intended as a starter foundation for building and explaining the ingestion framework in a clear, easy-to-understand way.
