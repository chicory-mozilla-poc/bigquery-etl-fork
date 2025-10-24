# Clients Daily v6

## Overview

The `clients_daily_v6` table is the primary daily aggregation of Firefox Desktop telemetry data. It provides one row per client per day, aggregating all main telemetry pings received from each Firefox Desktop client. This table is essential for understanding user behavior, product health, and feature adoption at daily granularity.

**Key Characteristics:**
- **Granularity:** One row per `client_id` per `submission_date`
- **Update Frequency:** Daily (part of the `bqetl_main_summary` DAG)
- **Data Retention:** 775 days (configurable via partition expiration)
- **Source Table:** `moz-fx-data-shared-prod.telemetry_stable.main_v5`
- **Primary Access:** Via user-facing view `telemetry.clients_daily`

## Table Schema

### Partitioning and Clustering

- **Partitioning Field:** `submission_date` (DAY granularity)
  - **Important:** Always include a `submission_date` filter in queries to avoid full table scans
  - Partition filter is **required** by the table configuration

- **Clustering Fields:**
  1. `normalized_channel` - Release channel (release, beta, nightly, esr)
  2. `sample_id` - Sampling bucket (0-99)

### Key Column Categories

The table contains 400+ columns organized into the following categories:

1. **Identifiers & Dimensions**
   - `client_id`, `sample_id`, `submission_date`
   - Geographic: `country`, `city`, `geo_subdivision1`, `geo_subdivision2`
   - System: `os`, `normalized_os_version`, `cpu_vendor`, `memory_mb`

2. **Engagement Metrics**
   - `active_hours_sum` - Active browsing time
   - `scalar_parent_browser_engagement_total_uri_count_sum` - Total page loads
   - `scalar_parent_browser_engagement_unique_domains_count_max` - Domain diversity
   - `sessions_started_on_this_day` - Session count

3. **Search Metrics**
   - `search_count_all` - Total searches
   - `search_counts` - Searches by engine and source (ARRAY of STRUCT)
   - `ad_clicks_count_all` - Ad click monetization
   - `search_with_ads_count_all` - SERP with ads impressions

4. **Stability & Performance**
   - `crashes_detected_content_sum`, `crash_submit_success_main_sum`
   - `aborts_content_sum`, `shutdown_kill_sum`
   - `subsession_hours_sum` - Total session time

5. **Feature Usage**
   - `active_addons` - Installed extensions (ARRAY of STRUCT)
   - `experiments` - Active experiments (ARRAY of STRUCT)
   - `devtools_toolbox_opened_count_sum` - Developer tools usage
   - `fxa_configured`, `sync_configured` - Account services

6. **Browser Configuration**
   - `e10s_enabled` - Multi-process mode
   - `is_default_browser` - Default browser status
   - `default_search_engine` - Search engine setting
   - User preferences (multiple `user_pref_*` fields)

## Data Sources

### Primary Source

```
moz-fx-data-shared-prod.telemetry_stable.main_v5
```

This table aggregates all `main` pings from Firefox Desktop clients where:
- `normalized_app_name = 'Firefox'`
- `document_id IS NOT NULL`
- Clients sending > 150,000 pings/day are excluded (overactive filter)

### Aggregation Logic

For each `client_id` and `submission_date` combination:

- **SUM aggregations:** Most count/metric fields (e.g., `search_count_*`, `crashes_*`)
- **AVG aggregations:** Mean values (e.g., `active_addons_count_mean`, `places_bookmarks_count_mean`)
- **MAX aggregations:** Maximum observed values (e.g., `scalar_parent_browser_engagement_max_concurrent_tab_count_max`)
- **mode_last aggregations:** Most recent value based on `submission_timestamp` (e.g., `normalized_channel`, `os`, most configuration fields)

## Update Schedule

- **DAG:** `bqetl_main_summary`
- **Schedule:** Daily
- **Start Date:** 2019-11-05
- **Typical Availability:** Data for date `D` is typically available by 06:00 UTC on `D+1`

### Downstream Dependencies

External tasks that depend on this table:
- `jetstream.wait_for_clients_daily` (2h delay)
- `operational_monitoring.wait_for_clients_daily` (2h delay)
- `parquet_export.wait_for_clients_daily` (1h delay)

## Usage Examples

### Example 1: Basic Engagement Analysis

```sql
SELECT
  submission_date,
  normalized_channel,
  COUNT(DISTINCT client_id) AS dau,
  AVG(active_hours_sum) AS avg_active_hours,
  SUM(scalar_parent_browser_engagement_total_uri_count_sum) AS total_uris
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date = '2024-01-15'
  AND normalized_channel = 'release'
  AND sample_id = 0  -- 1% sample
GROUP BY
  submission_date, normalized_channel
```

### Example 2: Search Behavior by Engine

```sql
SELECT
  submission_date,
  search.engine,
  search.source,
  COUNT(DISTINCT client_id) AS clients,
  SUM(search.count) AS total_searches
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`,
  UNNEST(search_counts) AS search
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-07'
  AND normalized_channel = 'release'
  AND sample_id < 10  -- 10% sample
GROUP BY
  submission_date, search.engine, search.source
ORDER BY
  total_searches DESC
```

### Example 3: Addon Usage Analysis

```sql
WITH addon_installs AS (
  SELECT
    submission_date,
    addon.addon_id,
    addon.name,
    COUNT(DISTINCT client_id) AS install_count
  FROM
    `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`,
    UNNEST(active_addons) AS addon
  WHERE
    submission_date = '2024-01-15'
    AND normalized_channel = 'release'
    AND sample_id = 0
    AND addon.is_system = FALSE  -- Exclude built-in addons
  GROUP BY
    submission_date, addon.addon_id, addon.name
)
SELECT
  *
FROM
  addon_installs
ORDER BY
  install_count DESC
LIMIT 20
```

### Example 4: Crash Rate Calculation

```sql
SELECT
  submission_date,
  normalized_channel,
  os,
  COUNT(DISTINCT client_id) AS total_clients,
  COUNTIF(crashes_detected_content_sum > 0) AS clients_with_crashes,
  SAFE_DIVIDE(
    COUNTIF(crashes_detected_content_sum > 0),
    COUNT(DISTINCT client_id)
  ) AS crash_rate,
  SUM(crashes_detected_content_sum) AS total_crashes
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
  AND normalized_channel = 'release'
  AND sample_id = 0
GROUP BY
  submission_date, normalized_channel, os
ORDER BY
  submission_date, crash_rate DESC
```

### Example 5: Feature Adoption Over Time

```sql
SELECT
  submission_date,
  COUNTIF(fxa_configured) AS fxa_enabled_clients,
  COUNTIF(sync_configured) AS sync_enabled_clients,
  COUNTIF(e10s_enabled) AS e10s_enabled_clients,
  COUNTIF(is_default_browser) AS default_browser_clients,
  COUNT(DISTINCT client_id) AS total_clients,
  SAFE_DIVIDE(COUNTIF(fxa_configured), COUNT(DISTINCT client_id)) AS fxa_adoption_rate
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
  AND normalized_channel = 'release'
  AND sample_id < 10  -- 10% sample
GROUP BY
  submission_date
ORDER BY
  submission_date
```

### Example 6: Cohort Retention Analysis

```sql
-- Identify new users on a specific date
WITH new_users AS (
  SELECT
    client_id,
    MIN(submission_date) AS first_seen_date
  FROM
    `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
  WHERE
    submission_date BETWEEN '2024-01-01' AND '2024-01-31'
    AND normalized_channel = 'release'
    AND sample_id = 0
  GROUP BY
    client_id
  HAVING
    first_seen_date = '2024-01-01'
),
-- Track their activity in subsequent days
activity AS (
  SELECT
    cd.client_id,
    cd.submission_date,
    DATE_DIFF(cd.submission_date, nu.first_seen_date, DAY) AS days_since_first_seen
  FROM
    `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6` cd
  INNER JOIN
    new_users nu
  ON
    cd.client_id = nu.client_id
  WHERE
    cd.submission_date BETWEEN '2024-01-01' AND '2024-01-31'
    AND cd.normalized_channel = 'release'
)
-- Calculate retention rates
SELECT
  days_since_first_seen,
  COUNT(DISTINCT client_id) AS active_users,
  SAFE_DIVIDE(
    COUNT(DISTINCT client_id),
    (SELECT COUNT(*) FROM new_users)
  ) AS retention_rate
FROM
  activity
GROUP BY
  days_since_first_seen
ORDER BY
  days_since_first_seen
```

## Performance Optimization Tips

### 1. Always Use Partition Filters

**Required:** Include `submission_date` filters to avoid scanning the entire table.

```sql
-- GOOD: Partition filter applied
WHERE submission_date = '2024-01-15'

-- GOOD: Range with partition filter
WHERE submission_date BETWEEN '2024-01-01' AND '2024-01-31'

-- BAD: No partition filter (will fail or scan entire table)
WHERE client_id = 'some-uuid'
```

### 2. Leverage Clustering

Use `normalized_channel` and `sample_id` in WHERE clauses:

```sql
-- GOOD: Uses clustering fields
WHERE
  submission_date = '2024-01-15'
  AND normalized_channel = 'release'
  AND sample_id < 10
```

### 3. Use Sample_id for Efficient Sampling

For quick analysis, use `sample_id` to sample data:

```sql
-- 1% sample
WHERE sample_id = 0

-- 10% sample
WHERE sample_id < 10

-- 50% sample
WHERE sample_id < 50
```

### 4. Select Only Needed Columns

Avoid `SELECT *` due to the large number of columns (400+):

```sql
-- GOOD: Select specific columns
SELECT client_id, active_hours_sum, search_count_all

-- BAD: Select all (expensive)
SELECT *
```

### 5. Pre-aggregate Before JOINs

When joining with other tables, pre-aggregate to reduce data volume:

```sql
WITH agg AS (
  SELECT
    client_id,
    SUM(active_hours_sum) AS total_hours
  FROM
    clients_daily_v6
  WHERE
    submission_date BETWEEN '2024-01-01' AND '2024-01-07'
  GROUP BY
    client_id
)
SELECT ...
FROM agg
JOIN other_table ...
```

## Related Tables

### Upstream Sources
- **`telemetry_stable.main_v5`** - Raw main ping data (source)

### Downstream Tables
- **`telemetry.clients_daily`** - User-facing view (recommended access point)
- **`telemetry_derived.clients_last_seen_v1`** - Longitudinal client tracking
- **`telemetry_derived.clients_last_seen_joined_v1`** - Enriched with event ping data

### Related Analysis Tables
- **`telemetry_derived.main_summary_v4`** - Legacy daily summary (deprecated, use clients_daily instead)
- **`telemetry_derived.clients_first_seen_v2`** - Client acquisition dates
- **`telemetry_derived.core_clients_first_seen_v1`** - First seen tracking

## Important Notes

### Data Quality Considerations

1. **Overactive Client Filtering:** Clients sending > 150,000 pings/day or with > 3M active addons are excluded to prevent aggregation errors.

2. **Mode_last Aggregation:** Many configuration fields use `mode_last` aggregation, which selects the most recent value based on `submission_timestamp`. This may not capture mid-day changes.

3. **Null Values:** Many fields can be NULL, especially for:
   - Metrics not collected on older Firefox versions
   - Optional telemetry probes
   - Platform-specific metrics (e.g., `windows_build_number` is NULL on non-Windows)

4. **Sampling Consistency:** `sample_id` is consistent across days for the same `client_id`, enabling cohort tracking.

### Schema Evolution

This is version 6 of the schema. Historical notes:
- Introduced 2019-11-05 (see `scheduling.start_date` in metadata)
- Depends on fields added to `main_v5` on 2019-11-22
- For data before 2019-11-22, use the legacy query version that read from `main_summary_v4`

### Privacy & Data Handling

- **PII:** This table contains pseudonymous identifiers (`client_id`) but no direct PII
- **Sampling:** Use `sample_id` for analysis requiring smaller samples
- **Retention:** Data is retained for 775 days before automatic deletion
- **Access:** Requires appropriate BigQuery dataset access permissions

## Contact & Support

- **Owner:** ascholtz@mozilla.com
- **DAG:** `bqetl_main_summary`
- **Labels:** `application:firefox`, `schedule:daily`, `table_type:client_level`
- **GitHub Issues:** [mozilla/bigquery-etl](https://github.com/mozilla/bigquery-etl/issues)
- **Documentation:** [docs.telemetry.mozilla.org](https://docs.telemetry.mozilla.org/)

## Change Log

### Version 6 (2019-11-05 - Present)
- Migrated from `main_summary_v4` to `main_v5` as source
- Added support for new addon fields (foreign_install, user_disabled, version)
- Added active_ticks and first_paint from simple_measurements
- Enhanced search metrics and Quick Suggest tracking
- Added migration quantity tracking (Chrome, Edge, Safari)
- Added media playback metrics
- Added numerous contextual services metrics

For the complete history and query evolution, see the [bigquery-etl repository](https://github.com/mozilla/bigquery-etl/blob/main/sql/moz-fx-data-shared-prod/telemetry_derived/clients_daily_v6/).