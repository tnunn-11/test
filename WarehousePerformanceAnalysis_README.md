# Warehouse Configuration Analysis — Account kn56889

_Generated: November 12, 2026 · Window: last 30 days · Mode: read-only (no changes applied)_

## Scope & Constraints
- Analysis and recommendations only. No warehouse, monitor, or account changes were executed.
- All recommended `ALTER WAREHOUSE` statements are online operations (no downtime). Running queries finish on existing compute.
- Resource monitor and `ALTER ACCOUNT` statements require `ACCOUNTADMIN`.

## Current Configuration

| Warehouse | Size | Gen | Clusters | Auto-suspend | Resource Monitor |
|---|---|---|---|---|---|
| ANALYST_WH | XS | 1 | 1–4 | 60s | None |
| COMPUTE_WH | XS | 1 | 1 | 60s | COMPUTE_WH_MONITOR (500) |
| DEVELOPER_WH | XS | 1 | 1 | 60s | DEVELOPER_WH_MONITOR (200) |
| FALCON_WH | XS | 1 | 1 | 60s | None |
| INGESTION_WH | XS | 1 | 1 | 60s | INGESTION_WH_MONITOR (100) |
| SNOWFLAKE_LEARNING_WH | XS | 1 | 1 | 300s | None |
| SYSTEM$STREAMLIT_NOTEBOOK_WH | XS | 1 | 1–10 | 60s | Monitor (no notify) |
| TRANSFORMATION_DYNAMICTABLE_WH | Large | 2 | 1 | 60s | None |
| TRANSFORMATION_WH | XS | 1 | 1 | 60s | TRANSFORMATION_WH_MONITOR (1000) |

`CREDITS_500` monitor exists but is not attached to the account or any warehouse.

## 30-Day Usage

| Warehouse | Credits | Billed hrs (/720) | Queries | p50 / p95 (s) | Queued queries | Spill local / remote | Avg running / queued |
|---|---|---|---|---|---|---|---|
| TRANSFORMATION_WH | 484.4 | 377 | 193,619 | 2.1 / 258 | 41,827 (22%) | 2,711 / 2 | 5.2 / 4.2 (peak 133) |
| COMPUTE_WH | 219.7 | 720 | 130,317 | 1.5 / 35 | 20,498 (16%) | 478 / 0 | 0.5 / 0.1 |
| ANALYST_WH | 183.9 | 241 | 44,277 | 2.2 / 385 | 4,454 | 2,438 / 0 | 4.6 / 1.3 |
| DEVELOPER_WH | 43.1 | 718 | 17,987 | 1.1 / 26 | 94 | 593 / 0 | 0.1 / 0.0 |
| TRANSFORMATION_DYNAMICTABLE_WH | 27.0 | 22 | 714 | 4.2 / 26 | 0 | 8 / 0 | 0.45 / 0 |
| FALCON_WH | 2.5 | 93 | 85 | 5.8 / 7 | 0 | 0 / 0 | ~0 |

Total ≈ 960 credits / 30 days.

## Findings
1. **TRANSFORMATION_WH is concurrency-bound.** Single-cluster XS; ~4 queries waiting per 5 running. Load is system `CALL`/`MERGE` (tasks/procedures).
2. **COMPUTE_WH and DEVELOPER_WH are billed ~24/7.** Auto-suspend never triggers.
   - COMPUTE_WH: steady trickle of `INSERT`/`MERGE`/`COPY` from `SVC_DATAENGINEERING` (~2.4/min).
   - DEVELOPER_WH: 14,260 dynamic-table refreshes — some dynamic tables use this warehouse with a short target lag.
3. **ANALYST_WH queries spill to disk** (2,438) with a 6+ minute p95 — XS is undersized for heavy `CALL`s.
4. **TRANSFORMATION_DYNAMICTABLE_WH (Large) is lightly used** — 22 billed hours, <1 concurrent query.
5. **Governance gaps:** no monitors on ANALYST_WH, FALCON_WH, SNOWFLAKE_LEARNING_WH, TRANSFORMATION_DYNAMICTABLE_WH; no account-level monitor; existing monitors **suspend at 100%** (downtime risk).
6. **Housekeeping:** INGESTION_WH unused since August; SNOWFLAKE_LEARNING_WH has 300s auto-suspend.

## Why Adaptive?
Snowflake's recommendation service lists FALCON_WH, DEVELOPER_WH, COMPUTE_WH, ANALYST_WH, TRANSFORMATION_WH as **Recommended** for Adaptive (based on expected performance gains with cost neutrality or reduction). Fit with observed data:
- **Per-query billing** → addresses 24/7 billing on COMPUTE_WH / DEVELOPER_WH.
- **Per-query sizing** → addresses spills and long p95 on ANALYST_WH.
- **Automatic concurrency scaling** → addresses the TRANSFORMATION_WH queue.
- Online conversion, reversible with `SET WAREHOUSE_TYPE = 'STANDARD'`. Resource monitors still apply.
- Results are not guaranteed — monitor after conversion.

## Recommendations (not executed)

### Option A — Adaptive (online, reversible)
```sql
ALTER WAREHOUSE TRANSFORMATION_WH SET WAREHOUSE_TYPE = 'ADAPTIVE';
ALTER WAREHOUSE ANALYST_WH        SET WAREHOUSE_TYPE = 'ADAPTIVE';
ALTER WAREHOUSE COMPUTE_WH        SET WAREHOUSE_TYPE = 'ADAPTIVE';
ALTER WAREHOUSE DEVELOPER_WH      SET WAREHOUSE_TYPE = 'ADAPTIVE';
ALTER WAREHOUSE FALCON_WH         SET WAREHOUSE_TYPE = 'ADAPTIVE';
-- Revert: ALTER WAREHOUSE <name> SET WAREHOUSE_TYPE = 'STANDARD';
```

### Option B — Stay Standard and tune (online; raises per-hour cost)
```sql
ALTER WAREHOUSE TRANSFORMATION_WH SET MAX_CLUSTER_COUNT = 3 SCALING_POLICY = 'STANDARD';
ALTER WAREHOUSE ANALYST_WH        SET WAREHOUSE_SIZE = 'SMALL';
ALTER WAREHOUSE TRANSFORMATION_DYNAMICTABLE_WH SET WAREHOUSE_SIZE = 'MEDIUM';
ALTER WAREHOUSE SNOWFLAKE_LEARNING_WH SET AUTO_SUSPEND = 60;

-- Find dynamic tables keeping DEVELOPER_WH awake (read-only)
SHOW DYNAMIC TABLES IN ACCOUNT;  -- filter warehouse = 'DEVELOPER_WH', review target_lag
```

### Resource Monitors — Notify Only (no suspend)
A monitor with only `NOTIFY` triggers never stops a warehouse. Alerts go to account admins who enabled notifications, or to users in `NOTIFY_USERS` (verified email required). `SET TRIGGERS` replaces all triggers.
```sql
USE ROLE ACCOUNTADMIN;

ALTER RESOURCE MONITOR COMPUTE_WH_MONITOR        SET TRIGGERS ON 50 PERCENT DO NOTIFY ON 80 PERCENT DO NOTIFY ON 100 PERCENT DO NOTIFY ON 125 PERCENT DO NOTIFY;
ALTER RESOURCE MONITOR TRANSFORMATION_WH_MONITOR SET TRIGGERS ON 50 PERCENT DO NOTIFY ON 80 PERCENT DO NOTIFY ON 100 PERCENT DO NOTIFY ON 125 PERCENT DO NOTIFY;
ALTER RESOURCE MONITOR DEVELOPER_WH_MONITOR      SET TRIGGERS ON 50 PERCENT DO NOTIFY ON 80 PERCENT DO NOTIFY ON 100 PERCENT DO NOTIFY;
ALTER RESOURCE MONITOR INGESTION_WH_MONITOR      SET TRIGGERS ON 80 PERCENT DO NOTIFY ON 100 PERCENT DO NOTIFY;
ALTER RESOURCE MONITOR SYSTEM$STREAMLIT_NOTEBOOK_WH_MONITOR SET TRIGGERS ON 80 PERCENT DO NOTIFY ON 100 PERCENT DO NOTIFY;

CREATE RESOURCE MONITOR IF NOT EXISTS ANALYST_WH_MONITOR WITH CREDIT_QUOTA = 250 FREQUENCY = MONTHLY START_TIMESTAMP = IMMEDIATELY
  TRIGGERS ON 50 PERCENT DO NOTIFY ON 80 PERCENT DO NOTIFY ON 100 PERCENT DO NOTIFY;
ALTER WAREHOUSE ANALYST_WH SET RESOURCE_MONITOR = ANALYST_WH_MONITOR;

CREATE RESOURCE MONITOR IF NOT EXISTS TRANSFORMATION_DT_WH_MONITOR WITH CREDIT_QUOTA = 100 FREQUENCY = MONTHLY START_TIMESTAMP = IMMEDIATELY
  TRIGGERS ON 50 PERCENT DO NOTIFY ON 80 PERCENT DO NOTIFY ON 100 PERCENT DO NOTIFY;
ALTER WAREHOUSE TRANSFORMATION_DYNAMICTABLE_WH SET RESOURCE_MONITOR = TRANSFORMATION_DT_WH_MONITOR;

ALTER RESOURCE MONITOR CREDITS_500 SET CREDIT_QUOTA = 1200
  TRIGGERS ON 75 PERCENT DO NOTIFY ON 100 PERCENT DO NOTIFY;
ALTER ACCOUNT SET RESOURCE_MONITOR = CREDITS_500;
```
Quotas on new monitors are placeholders based on 30-day usage. Snowflake Budgets is an alternative for dollar-based notify-only alerts.

## Analysis Queries (read-only)
```sql
-- 30-day credits and query performance
WITH m AS (
  SELECT warehouse_name, SUM(credits_used) credits, COUNT(DISTINCT DATE_TRUNC('hour',start_time)) active_hrs
  FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
  WHERE start_time >= DATEADD(day,-30,CURRENT_TIMESTAMP()) AND credits_used > 0 GROUP BY 1),
q AS (
  SELECT warehouse_name, COUNT(*) queries,
    ROUND(MEDIAN(total_elapsed_time)/1000,2) p50_s,
    ROUND(APPROX_PERCENTILE(total_elapsed_time,0.95)/1000,1) p95_s,
    SUM(IFF(queued_overload_time>0,1,0)) queued_q,
    SUM(IFF(bytes_spilled_to_local_storage>0,1,0)) local_spill_q,
    SUM(IFF(bytes_spilled_to_remote_storage>0,1,0)) remote_spill_q,
    ROUND(SUM(execution_time)/3600000,2) exec_hrs
  FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE start_time >= DATEADD(day,-30,CURRENT_TIMESTAMP()) AND warehouse_name IS NOT NULL AND warehouse_size IS NOT NULL GROUP BY 1)
SELECT COALESCE(m.warehouse_name,q.warehouse_name) wh, ROUND(credits,2) credits, active_hrs,
  queries, p50_s, p95_s, queued_q, local_spill_q, remote_spill_q, exec_hrs
FROM m FULL OUTER JOIN q ON m.warehouse_name = q.warehouse_name ORDER BY credits DESC NULLS LAST;

-- Concurrency / queue load
SELECT warehouse_name, ROUND(AVG(avg_running),2) avg_running, ROUND(MAX(avg_running),2) peak_running,
  ROUND(AVG(avg_queued_load),2) avg_queued, ROUND(MAX(avg_queued_load),2) peak_queued
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE start_time >= DATEADD(day,-30,CURRENT_TIMESTAMP()) AND warehouse_name NOT ILIKE 'COMPUTE_SERVICE%'
GROUP BY 1 ORDER BY avg_queued DESC;

-- Configuration
SHOW WAREHOUSES LIKE '%';
SHOW RESOURCE MONITORS;
```
