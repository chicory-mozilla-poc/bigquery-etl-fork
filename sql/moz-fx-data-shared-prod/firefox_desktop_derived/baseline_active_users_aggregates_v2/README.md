# Baseline Active Users Aggregates v2

## Overview

The `baseline_active_users_aggregates_v2` table provides daily aggregated counts of Firefox Desktop active users across multiple dimensions including geography, application version, operating system, channel, and user attributes. This table is designed for efficient querying and analysis of user engagement metrics by pre-aggregating data from the client-level `baseline_active_users` table.

**Key Features:**
- Daily granularity with date-based partitioning on `submission_date`
- Pre-aggregated user counts for DAU, WAU, and MAU metrics
- Multi-dimensional breakdowns (20+ grouping dimensions)
- Optimized for reporting and dashboards through clustering on app_name, channel, and country
- Incremental refresh pattern for efficient daily updates

**Primary Use Cases:**
- Calculating and tracking Daily Active Users (DAU), Weekly Active Users (WAU), and Monthly Active Users (MAU)
- Geographic analysis of Firefox Desktop usage patterns
- Version adoption tracking and release impact analysis
- Channel-specific performance monitoring (release, beta, nightly, ESR)
- Attribution and acquisition funnel analysis
- OS compatibility and platform distribution analysis

## Table-Level Quality Description

This table aggregates Firefox Desktop baseline telemetry data to provide pre-calculated active user counts segmented by key dimensions. It transforms client-level activity flags from the `baseline_active_users` view into aggregated metrics by grouping on 20 dimensions including temporal (submission_date, first_seen_year), geographic (country, city), technical (OS, app version), and attribution fields.

The aggregation logic uses `COUNTIF` on boolean activity flags to calculate six distinct metrics: daily_users, weekly_users, monthly_users, dau, wau, and mau. The distinction between "users" metrics (based on ping submission) and active user metrics (based on activity criteria) enables nuanced analysis of engagement patterns.

This is a derived table that runs daily as part of the `bqetl_analytics_aggregations` DAG, processing data for the parameterized submission_date. The table uses day-based time partitioning with required partition filters and clustering on app_name, channel, and country for optimal query performance. Owned by the Analytics team, this table serves as a foundational dataset for Firefox Desktop KPI reporting and executive dashboards.

## Schema

The table contains 32 fields organized into the following categories:

### Temporal Dimensions
- `submission_date` (DATE): Partition key representing ping receipt date
- `first_seen_year` (INTEGER): Year of client's first appearance

### Geographic Dimensions
- `country` (STRING): ISO country code (clustering key)
- `city` (STRING): City name from IP geolocation
- `locale` (STRING): Language/country preference (e.g., en-US)

### Application Dimensions
- `app_name` (STRING): Application name identifier (clustering key)
- `app_version` (STRING): User-visible version string
- `app_version_major` (NUMERIC): Major version number
- `app_version_minor` (NUMERIC): Minor version number
- `app_version_patch_revision` (NUMERIC): Patch/revision number
- `app_version_is_major_release` (BOOLEAN): Major release flag
- `channel` (STRING): Release channel (clustering key)
- `distribution_id` (STRING): Distribution identifier

### Operating System Dimensions
- `os` (STRING): Operating system name
- `os_grouped` (STRING): OS category grouping
- `os_version` (STRING): Full OS version string
- `os_version_major` (INTEGER): OS major version
- `os_version_minor` (INTEGER): OS minor version
- `os_version_build` (STRING): OS build number
- `windows_build_number` (INTEGER): Windows-specific build

### User Attribute Dimensions
- `is_default_browser` (BOOLEAN): Default browser status
- `activity_segment` (STRING): User engagement classification
- `attribution_medium` (STRING): Acquisition medium
- `attribution_source` (STRING): Acquisition source

### Metrics (Aggregated Counts)
- `daily_users` (INTEGER): Count of users submitting pings on submission_date
- `weekly_users` (INTEGER): Count of users submitting pings in 7-day window
- `monthly_users` (INTEGER): Count of users submitting pings in 28-day window
- `dau` (INTEGER): Count of daily active users (activity-based)
- `wau` (INTEGER): Count of weekly active users (activity-based)
- `mau` (INTEGER): Count of monthly active users (activity-based)

For complete field-level descriptions with context and derivation details, see `schema.yaml`.

## Data Sources

### Primary Source
- **Table**: `moz-fx-data-shared-prod.firefox_desktop.baseline_active_users`
- **Type**: View (client-level active users data)
- **Relationship**: This derived table aggregates the source view by grouping on dimension columns and counting activity flags

### Upstream Dependencies
The `baseline_active_users` view itself is built from Firefox Desktop baseline ping telemetry, incorporating:
- Raw baseline ping data with client identifiers and activity metrics
- Activity classification logic (is_dau, is_wau, is_mau flags)
- User classification logic (is_daily_user, is_weekly_user, is_monthly_user flags)
- Normalized dimension fields (country, locale, OS, app version)
- Attribution data from install and acquisition tracking

## Update Schedule

- **Frequency**: Daily
- **DAG**: `bqetl_analytics_aggregations`
- **Execution Time**: Scheduled as part of the analytics aggregations workflow
- **Incremental Update**: Yes (processes only the parameterized submission_date)
- **Partition Strategy**: Day-based partitioning on `submission_date` with required partition filter
- **Latency**: Data becomes available after the daily analytics aggregations DAG completes

**Owners:**
- kwindau@mozilla.com
- ago@mozilla.com

## Usage Examples

### Example 1: Calculate Global MAU for a Specific Date
```sql
SELECT
  submission_date,
  SUM(mau) AS total_mau
FROM
  `moz-fx-data-shared-prod.firefox_desktop_derived.baseline_active_users_aggregates_v2`
WHERE
  submission_date = '2024-01-15'
GROUP BY
  submission_date
```

### Example 2: DAU by Country and Channel
```sql
SELECT
  submission_date,
  country,
  channel,
  SUM(dau) AS daily_active_users
FROM
  `moz-fx-data-shared-prod.firefox_desktop_derived.baseline_active_users_aggregates_v2`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
  AND country IN ('US', 'DE', 'FR', 'GB', 'CA')
  AND channel = 'release'
GROUP BY
  submission_date,
  country,
  channel
ORDER BY
  submission_date,
  daily_active_users DESC
```

### Example 3: Version Adoption Analysis
```sql
SELECT
  submission_date,
  app_version_major,
  channel,
  SUM(mau) AS monthly_active_users
FROM
  `moz-fx-data-shared-prod.firefox_desktop_derived.baseline_active_users_aggregates_v2`
WHERE
  submission_date = '2024-01-31'
  AND channel IN ('release', 'beta')
GROUP BY
  submission_date,
  app_version_major,
  channel
ORDER BY
  channel,
  app_version_major DESC
```

### Example 4: Activity Segment Distribution
```sql
SELECT
  submission_date,
  activity_segment,
  SUM(dau) AS daily_active_users,
  SUM(wau) AS weekly_active_users,
  SUM(mau) AS monthly_active_users
FROM
  `moz-fx-data-shared-prod.firefox_desktop_derived.baseline_active_users_aggregates_v2`
WHERE
  submission_date = '2024-01-15'
  AND activity_segment IS NOT NULL
GROUP BY
  submission_date,
  activity_segment
ORDER BY
  monthly_active_users DESC
```

### Example 5: New Users (First Year Cohort) by Attribution
```sql
SELECT
  submission_date,
  first_seen_year,
  attribution_source,
  attribution_medium,
  SUM(dau) AS new_cohort_dau,
  SUM(mau) AS new_cohort_mau
FROM
  `moz-fx-data-shared-prod.firefox_desktop_derived.baseline_active_users_aggregates_v2`
WHERE
  submission_date = '2024-01-15'
  AND first_seen_year = 2024
  AND attribution_source IS NOT NULL
GROUP BY
  submission_date,
  first_seen_year,
  attribution_source,
  attribution_medium
ORDER BY
  new_cohort_mau DESC
```

## Related Tables

### Upstream
- `moz-fx-data-shared-prod.firefox_desktop.baseline_active_users` - Source view with client-level activity flags
- `moz-fx-data-shared-prod.firefox_desktop_derived.baseline_active_users_aggregates_v1` - Previous version of this aggregation

### Related Tables
- `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6` - Alternative source for desktop user metrics based on main ping
- `moz-fx-data-shared-prod.telemetry.clients_last_seen` - Client-level longitudinal activity tracking
- `moz-fx-data-shared-prod.firefox_desktop.baseline_active_users_aggregates` - Public view of this table

### Downstream
- Executive dashboards and KPI reporting tools
- Growth and retention analysis pipelines
- Attribution and marketing effectiveness reports

## Notes

### Performance Optimization
- **Always include a partition filter** on `submission_date` - this is required by the table definition
- Leverage clustering keys (app_name, channel, country) in WHERE clauses for optimal query performance
- Pre-aggregated nature makes this table ideal for reporting; avoid re-aggregating at client level

### Metric Definitions
- **"Users" metrics** (daily_users, weekly_users, monthly_users): Based on ping submission behavior
- **"Active Users" metrics** (dau, wau, mau): Based on activity criteria (e.g., URI visits, active ticks)
- The difference enables analysis of "seen but not active" vs. "actively engaged" users

### Data Quality Considerations
- Unknown or NULL country values are represented as '??'
- Some dimensional attributes may be NULL for certain client segments
- App_name field may be modified for MozillaOnline and BrowserStack distributions
- Windows_build_number is NULL for non-Windows platforms

### Schema Evolution
- This is version 2 (v2) of the aggregates table
- Version changes may indicate schema modifications or calculation methodology updates
- Column descriptions are maintained separately in schema.yaml

### Access Patterns
- Optimized for aggregation queries with dimensional filtering
- Not suitable for client-level analysis (use source table instead)
- Best used for time-series analysis, cohort tracking, and segment comparisons

---

**Documentation Generated**: 2024
**Table Version**: v2
**Schema Definitions**: See `schema.yaml` for complete field specifications
