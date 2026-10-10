# Gateway Lifecycle Control for Metadata-Driven Ingestion

## Executive Summary

The metadata-driven ingestion model is most efficient when each source gateway is active only during the actual ingestion window and inactive at all other times.

This avoids a common operational problem: a gateway that stays on 24x7 creates unnecessary cost, idle connections, and unmanaged CDC activity.

The recommended pattern is:

- one gateway per database instance
- metadata-driven routing of tables to that gateway
- gateway activation only during the scheduled ingestion window
- automatic deactivation immediately after the run completes

This enables Cuscal to keep the architecture scalable while controlling cloud cost and operational overhead.

---

## Why Gateway Control Matters

A gateway is not a table-specific job. It is a shared source connection that handles:

- database connectivity
- CDC capture / incremental change polling
- secure credentials and network access
- event streaming across many tables in the same source instance

If that gateway remains active all the time, cost accumulates even when no ingestion is running.

### Example Cost Problem

If 10 gateways are kept live 24x7 and each costs $50 per month, the annual cost is:

```
10 gateways × $50 × 12 = $6,000 / year
```

If those gateways are only active for 1 hour per day, the cost can fall to a small fraction of that amount.

---

## Design Principle

The framework should follow this lifecycle pattern:

```
Idle state → scheduled activation → ingestion run → post-run validation → gateway shutdown
```

This means a gateway is treated like a short-lived operational resource, not a permanently running service.

---

## Gateway Lifecycle Pattern

### State Model

Each gateway should have well-defined lifecycle states:

- `INACTIVE`
- `PENDING_ACTIVATION`
- `ACTIVE`
- `RUNNING`
- `DEACTIVATING`
- `FAILED`
- `ERROR`

### Recommended Behavior

```
At 1:55 AM:
  validate credentials and source health
  check whether run should start
  queue gateway activation

At 2:00 AM:
  activate gateway
  start CDC subscription or polling
  start metadata-driven ingestion tasks

At 2:55 AM:
  complete remaining table loads
  reconcile run status and audit entries

At 3:00 AM:
  stop CDC subscriptions
  shut down gateway
  store final run metrics
```

This keeps the gateway only active when there is actual work to do.

---

## Metadata Model for Gateway Control

A separate metadata table can govern gateway activation schedules.

```sql
CREATE TABLE IF NOT EXISTS gateway_control_config (
    gateway_id STRING,
    gateway_name STRING,
    source_system STRING,
    activation_type STRING,         -- SCHEDULED, ON_DEMAND, ALWAYS_ON
    schedule_cron STRING,           -- e.g. '0 2 * * *'
    activation_lead_time_min INT,  -- e.g. 5 minutes before run starts
    deactivation_delay_min INT,     -- e.g. 5 minutes after completion
    max_active_duration_min INT,    -- e.g. 120 minutes
    active_flag STRING,             -- Y/N
    last_activated TIMESTAMP,
    last_deactivated TIMESTAMP,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

### Example Records

```sql
INSERT INTO gateway_control_config VALUES (
    'sqlserver_erp_conn',
    'ERP SQL Server Gateway',
    'SQL Server',
    'SCHEDULED',
    '0 2 * * *',
    5,
    5,
    120,
    'Y',
    NULL,
    NULL,
    current_timestamp(),
    current_timestamp()
);

INSERT INTO gateway_control_config VALUES (
    'oracle_apps_conn',
    'Oracle Applications Gateway',
    'Oracle',
    'SCHEDULED',
    '0 2 * * *',
    5,
    5,
    120,
    'Y',
    NULL,
    NULL,
    current_timestamp(),
    current_timestamp()
);
```

This allows the framework to decide, at run time, which gateways should be active for a given schedule.

---

## How the Framework Activates and Deactivates Gateways

### Activation Workflow

At the start of the scheduled run, the framework should:

1. Read the gateway configuration table
2. Identify all gateways with `activation_type = 'SCHEDULED'`
3. Check if the current time is within the activation window
4. Validate source connectivity and credentials
5. Start the Lakeflow gateway connection
6. Start CDC subscriptions or polling for that database instance
7. Trigger metadata-based table processing for all tables mapped to that gateway

### Deactivation Workflow

After processing completes:

1. Confirm ingestion success or failure status for all tables in the gateway
2. Wait for the configured deactivation delay
3. Stop CDC subscriptions or polling
4. Close the gateway connection
5. Record final status in the gateway audit table

### Example Pseudocode

```python
# Pseudocode for gateway lifecycle management

def manage_gateway_lifecycle():
    gateways = read_gateway_control_config()
    
    for gateway in gateways:
        if gateway.activation_type == 'SCHEDULED' and should_activate(gateway):
            activate_gateway(gateway)
            continue

        if gateway.activation_type == 'ON_DEMAND' and manual_run_requested(gateway):
            activate_gateway(gateway)
            continue

        if gateway.activation_type == 'ALWAYS_ON':
            keep_gateway_active(gateway)
            continue

    # handle deactivation after run
    for gateway in active_gateways():
        if run_complete_for_gateway(gateway):
            deactivate_gateway(gateway)
```

---

## Example Runtime Sequence

### Daily Schedule Example

```
1:55 AM  -> framework starts
1:59 AM  -> validate connection credentials
2:00 AM  -> activate sqlserver_erp_conn
2:00 AM  -> activate oracle_apps_conn
2:00 AM  -> start metadata-driven processing for all mapped tables
2:55 AM  -> complete final loads and validation
3:00 AM  -> deactivate sqlserver_erp_conn
3:00 AM  -> deactivate oracle_apps_conn
3:05 AM  -> generate run summary and audit records
```

This ensures the gateways are only present during active ingestion.

---

## Why This Is Important for Cuscal

This pattern helps Cuscal achieve:

- reduced compute and connector cost
- lower risk of idle infrastructure spending
- predictable operational windows for ingestion
- better alignment with batch schedules and SLA windows
- cleaner control over connection lifecycle and failure isolation

In practice, the gateway is no longer a permanent infrastructure object. It becomes a scheduled runtime component.

---

## Gateway Control Audit Table

A separate audit table tracks gateway lifecycle events:

```sql
CREATE TABLE IF NOT EXISTS gateway_control_audit (
    event_id STRING,
    gateway_id STRING,
    event_type STRING,        -- ACTIVATED, DEACTIVATED, FAILED, TIMEOUT
    event_timestamp TIMESTAMP,
    event_status STRING,      -- SUCCESS, FAILED, WARNING
    error_message STRING,
    duration_seconds INT,
    active_table_count INT
);
```

### Example data

```sql
INSERT INTO gateway_control_audit VALUES (
    'evt_1001',
    'sqlserver_erp_conn',
    'ACTIVATED',
    current_timestamp(),
    'SUCCESS',
    NULL,
    30,
    50
);

INSERT INTO gateway_control_audit VALUES (
    'evt_1002',
    'sqlserver_erp_conn',
    'DEACTIVATED',
    current_timestamp(),
    'SUCCESS',
    NULL,
    300,
    50
);
```

This creates an operational record that proves the gateway was active only during the run window.

---

## DAB and GitHub Actions Orchestration Pattern

The lifecycle can be controlled from the deployment orchestration layer:

### Example DAB Pattern

```yaml
jobs:
  - name: gateway_lifecycle_ingestion
    tasks:
      - task_key: pre_run_validation
        spark_python_task:
          python_file: assets/pre_run_validation.py

      - task_key: activate_gateways
        depends_on:
          - task_key: pre_run_validation
        spark_python_task:
          python_file: assets/activate_gateways.py

      - task_key: process_metadata_ingestion
        depends_on:
          - task_key: activate_gateways
        spark_python_task:
          python_file: assets/process_metadata_ingestion.py

      - task_key: deactivate_gateways
        depends_on:
          - task_key: process_metadata_ingestion
        spark_python_task:
          python_file: assets/deactivate_gateways.py
```

### GitHub Actions Example

```yaml
name: Start Scheduled Ingestion

on:
  schedule:
    - cron: '0 2 * * *'

jobs:
  ingest:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Trigger Databricks orchestration
        run: |
          databricks bundle deploy
          databricks jobs run-now --job-id ${{ secrets.DATABRICKS_JOB_ID }}
```

This creates a clean scheduled lifecycle without requiring gateways to stay active continuously.

---

## Recommended Control Policy

### Default Policy

For enterprise ingestion workloads, the default should be:

- `SCHEDULED` for all source gateways
- one activation window per day or per batch window
- activation only within the job schedule
- immediate deactivation after processing completes

### Exceptions

Use `ALWAYS_ON` only when:

- the source must be continuously available for operational reporting
- a near-real-time streaming pattern is required
- there is a valid business reason to keep a connection open all the time

Use `ON_DEMAND` for:

- manual testing
- troubleshooting
- isolated recovery or partial reprocessing

---

## Operational Guidance

### Good Practice

- Treat gateway activation as a controlled runtime action, not a permanent configuration
- Always include deactivation in the pipeline as a final task
- Log every activation and deactivation event for auditability
- Add timeout guards to prevent a stuck gateway from remaining active forever
- Use metadata-driven schedule definitions so there is no hard-coded logic in each job

### Anti-Pattern to Avoid

Do not allow a gateway to remain active indefinitely without an explicit shutdown step. This creates:

- unnecessary cost
- idle connections
- stale CDC subscriptions
- operational confusion
- uncontrolled security exposure

---

## Cost Impact Summary

If gateways are active only during the ingestion run:

- compute and connector usage is reduced dramatically
- connection capacity is consumed only when needed
- the architecture remains scalable and operationally controlled
- budget is preserved for actual ingestion rather than idle connectivity

### Illustrative Example

```text
Always-on gateways:
- 7 gateways × 24 hours/day × 30 days = 5,040 active gateway-hours/month

Scheduled gateways:
- 7 gateways × 1 hour/day × 30 days = 210 active gateway-hours/month

Savings:
- 5,040 / 210 = 24x reduction in active gateway time
- equivalent cost reduction of ~95-96% in idle runtime
```

This is a strong operational and financial argument for dynamic gateway lifecycle control.

---

## Recommended Final Pattern

For Cuscal, the most practical pattern is:

- one gateway per database instance
- all tables in that instance use the same gateway metadata
- gateways are activated only during the scheduled ingestion run
- gateways are deactivated automatically immediately after completion
- lifecycle actions are fully recorded in audit tables

This gives the enterprise both cost discipline and architectural simplicity.

---

## Conclusion

The metadata-driven framework becomes stronger when source gateways are managed as finite runtime resources.

This allows Cuscal to retain the benefits of shared connectivity, centralized CDC, and table-level metadata control while avoiding the major downside of having every gateway always running.

The right operating model is:

> activate on schedule, process tables, validate results, deactivate immediately.

This is the cleanest way to balance operational performance, cost discipline, and enterprise-scale ingestion governance.

---

## Optional Next Step

The next practical step is to add a `gateway_control_config` table to the repository and define a small prototype workflow demonstrating:

- gateway activation at 2:00 AM
- metadata-driven table processing
- final deactivation at 3:00 AM
- gateway audit logging

This would convert the design concept into an implementable POC artifact.
