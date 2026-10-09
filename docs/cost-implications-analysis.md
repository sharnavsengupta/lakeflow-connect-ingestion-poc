# Cost Implications: Metadata-Driven Ingestion with Lakeflow Connect vs. Traditional Approach

## Detailed Cost Comparison

### Traditional Approach: RDS-Based Metadata + Databricks Custom Scripts

**Ingestion Architecture:**
- RDS metadata repository (AWS hosted)
- 500 custom Python/PySpark scripts (1 script per table)
- Databricks clusters for script execution (sequential or loosely parallelized)
- Manual error handling and retry logic

**Annual Costs Breakdown:**

| Cost Component | Details | Annual Cost |
|---|---|---|
| **RDS Instance** | Multi-AZ db.r6i.2xlarge (60 GB memory) | $12,000 |
| **Databricks Compute** | r5.4xlarge all-purpose cluster × 10 hrs/day × 365 days | $280,000 |
| **Databricks Storage** | DBFS + Delta tables (900 TB) | $0 |
| **Data Transfer** | Cross-region/VPC egress (900 TB/year @ $0.02/GB) | $18,000 |
| **Custom Script Maintenance** | Developer time: 500 scripts × 8 hrs/year × $150/hr | $600,000 |
| **Monitoring & Logging** | Databricks SQL, dashboards, custom logging | $10,000 |
| **Backup & Disaster Recovery** | RDS backups, data replication | $15,000 |
| **Support & Incidents** | Databricks support, incident debugging (75 hrs/year × $150/hr) | $11,250 |
| **License Fees** | Data movement tools, custom adapters | $10,000 |
| **Training & Documentation** | Onboarding, framework maintenance | $25,000 |
| **Estimated Wasted Compute** | Failed jobs, retries, redundant processing (~15%) | $42,000 |
| **TOTAL ANNUAL COST** | | **$1,023,250** |

---

### Metadata-Driven Approach: Lakeflow Connect + Databricks

**Ingestion Architecture:**
- Databricks Lakehouse (Unity Catalog)
- Lakeflow Connect connectors for multiple sources
- Single metadata-driven framework (one codebase for all 500 tables)
- Automated error isolation and retries

**Annual Costs Breakdown:**

| Cost Component | Details | Annual Cost |
|---|---|---|
| **Databricks Compute** | r5.4xlarge all-purpose cluster × 1 hr/day × 365 days | $28,000 |
| **Databricks Storage** | DBFS + Delta tables (900 TB @ inclusive pricing) | $0 |
| **Lakeflow Connect** | Included in Databricks Premium (no additional cost) | $0 |
| **GitHub Actions** | Standard GitHub Enterprise (already in use) | $0 |
| **Unity Catalog Governance** | Included in Databricks Premium | $0 |
| **Data Transfer** | Ingestion from external sources (minimal overhead) | $3,200 |
| **Framework Maintenance** | Developer time: 1 FTE × 20% allocation × $150/hr | $15,600 |
| **Monitoring & Logging** | Databricks SQL, Lakeflow metrics (built-in) | $0 |
| **Backup & DR** | Databricks managed backup (included) | $0 |
| **Support & Incidents** | Databricks support + isolated incident resolution (20 hrs/year) | $3,000 |
| **Training** | One-time framework training (amortized $2K/yr) | $2,000 |
| **TOTAL ANNUAL COST** | | **$51,800** |

---

## DBU Consumption Comparison

### Traditional Approach: Sequential Script Execution

**Scenario:** 500 tables × ~1 hour per script (sequential or loosely parallelized)

- **Compute requirement:** 10 hours/day × 365 days/year = 3,650 hours/year
- **r5.4xlarge cluster DBU rate:** ~4 DBUs/hour
- **Annual DBU consumption:** 3,650 hours × 4 DBUs = **14,600 DBUs/year**
- **Per run DBU cost:** ~40 DBUs per daily ingestion cycle

### Metadata-Driven Approach: Parallel Execution

**Scenario:** 500 tables processed concurrently in single run

- **Compute requirement:** 1 hour/day × 365 days/year = 365 hours/year
- **r5.4xlarge cluster DBU rate:** ~4 DBUs/hour
- **Annual DBU consumption:** 365 hours × 4 DBUs = **1,460 DBUs/year**
- **Per run DBU cost:** ~4 DBUs per daily ingestion cycle

**DBU Savings:** 14,600 - 1,460 = **13,140 DBUs/year (90% reduction)**

---

## Cost Savings Analysis

### Direct Ingestion Cost Savings

**Annual Savings:** $1,023,250 - $51,800 = **$971,450**

**Per-Table Cost Reduction:**
- Traditional: $1,023,250 ÷ 500 tables = **$2,047/table/year**
- Metadata-Driven: $51,800 ÷ 500 tables = **$104/table/year**
- **Savings per table: $1,943/year (95% reduction)**

### Cost Breakdown by Category

| Cost Category | Traditional | Metadata-Driven | Savings |
|---|---|---|---|
| Databricks Compute | $280,000 | $28,000 | $252,000 |
| Data Transfer | $18,000 | $3,200 | $14,800 |
| RDS Maintenance | $12,000 | $0 | $12,000 |
| **Script Maintenance** | **$600,000** | **$15,600** | **$584,400** |
| Wasted Compute/Retries | $42,000 | $0 | $42,000 |
| Monitoring, support, misc. | $71,250 | $5,000 | $66,250 |
| **TOTAL** | **$1,023,250** | **$51,800** | **$971,450** |

---

## Key Advantages of Metadata-Driven Approach

### 1. **Massive Compute Reduction**
- Traditional: 10 hrs/day × 365 days = 3,650 hours/year (14,600 DBUs)
- Metadata-driven: 1 hr/day × 365 days = 365 hours/year (1,460 DBUs)
- **Result:** 90% fewer DBUs consumed

### 2. **Cost Elimination**
- RDS instance eliminated ($12K/year)
- Custom script maintenance eliminated ($600K/year)
- Wasted compute from retries/failures eliminated ($42K/year)

### 3. **Scalability Without Proportional Cost**
- Traditional: Add 50% more tables → Need 50% more developer time + 50% more compute
- Metadata-driven: Add 50% more tables → Minimal additional cost (metadata only, same compute)

### 4. **Reduced Incidents**
- Traditional: 8-10 incidents/month (scattered scripts, inconsistent error handling)
- Metadata-driven: 2-3 incidents/month (framework-level, isolated failures)
- **Result:** 75% fewer support hours, less wasted compute from failed runs

### 5. **Lakeflow Connect Advantages**
- Handles incremental loads, CDC, parallel extraction natively
- No need to write custom connectors per source
- Built-in retry logic and error handling
- Metadata-driven configuration per source (no code changes)

---

## Financial Summary

| Timeframe | Traditional Approach | Metadata-Driven Approach | Total Savings |
|---|---|---|---|
| **Monthly** | $85,271 | $4,317 | $80,954 |
| **Quarterly** | $255,813 | $12,950 | $242,863 |
| **Annually** | $1,023,250 | $51,800 | $971,450 |
| **5-Year** | $5,116,250 | $259,000 | $4,857,250 |

---

## Recommendation

**Proceed with metadata-driven ingestion using Lakeflow Connect.**

The core financial case is straightforward: by consolidating 500 custom scripts into a single metadata-driven framework, Cuscal reduces annual ingestion costs by **$971K** (95% reduction per table) while simultaneously reducing DBU consumption by 90%, enabling faster ingestion, and eliminating maintenance overhead.
