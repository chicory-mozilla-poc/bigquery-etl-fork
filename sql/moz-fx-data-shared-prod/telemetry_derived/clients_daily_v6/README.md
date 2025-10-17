# Clients Daily v6

## Overview

`clients_daily_v6` is a core telemetry table that provides a **daily aggregation of Firefox desktop client activity metrics**. This table consolidates multiple `main` ping submissions from each client into a single row per client per day, making it the primary dataset for analyzing Firefox desktop user behavior, feature adoption, and product health metrics.

**Table**: `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
**Public View**: `moz-fx-data-shared-prod.telemetry.clients_daily`

> **Note**: As of Q1 2021, the public view `telemetry.clients_daily` references the downstream table `clients_last_seen_joined_v1` which merges additional fields from the `event` ping. See [Issue #1761](https://github.com/mozilla/bigquery-etl/issues/1761) for more details.

## Table-Level Quality Description

### Purpose
This table aggregates daily Firefox desktop telemetry data from the `main` ping, providing a comprehensive view of client-level metrics including:
- **Engagement metrics**: Active hours, URI counts, tab/window activity
- **System information**: OS, CPU, memory, graphics card details
- **Browser configuration**: Add-ons, search engines, preferences, experiments
- **Stability metrics**: Crashes, hangs, and error rates
- **Feature usage**: Developer tools, search activity, contextual services interactions
- **Sync and services**: Firefox Account status, Sync device counts

### Data Sources and Dependencies

**Primary Source**: `moz-fx-data-shared-prod.telemetry_stable.main_v5`
- Filters for `normalized_app_name = 'Firefox'` (desktop only)
- Excludes rows where `document_id IS NULL`
- Filters out overactive clients (>150,000 pings/day or >3,000,000 total addons)

**ETL Logic**:
1. **Base Processing**: Extracts and preprocesses histogram and scalar metrics from main pings
2. **Overactive Client Filtering**: Identifies and excludes clients with abnormal ping volumes to prevent aggregation errors
3. **Client-Level Aggregation**: Groups pings by `client_id` and `submission_date`, applying various aggregation strategies:
   - `SUM()` for counters and event counts
   - `AVG()` for means (active hours, clock skew, etc.)
   - `mode_last()` for configuration values (takes most recent mode)
   - `ARRAY_CONCAT_AGG()` for complex structures like add-ons

### Update Schedule

- **Frequency**: Daily
- **DAG**: `bqetl_main_summary`
- **Start Date**: November 5, 2019
- **Partition Field**: `submission_date` (DATE)
- **Partition Expiration**: 775 days (~2.1 years)
- **Clustering**: `normalized_channel`, `sample_id`

**External Dependencies**:
- Jetstream (wait_for_clients_daily, +2h execution delta)
- Operational Monitoring (wait_for_clients_daily, +2h execution delta)
- Parquet Export (wait_for_clients_daily, +1h execution delta)

### Data Characteristics

**Volume**: Millions of rows per day (one row per active Firefox desktop client)

**Key Features**:
- **Partition filtering required**: Queries must include a `submission_date` filter
- **Sampling**: Use `sample_id` (0-99) for representative data sampling
- **Client identification**: Each row represents one client's aggregated activity for one day
- **Null handling**: Many metrics can be NULL if not reported in any ping for that day

**Quality Considerations**:
- Clients with abnormally high ping volumes are excluded to maintain data quality
- Aggregation uses "mode_last" strategy for configuration values, prioritizing most recent state
- Histogram values are extracted and aggregated using specialized UDFs
- Geographic data (country, city, subdivisions) derived from IP geolocation

## Schema

The table contains 300+ columns organized into the following categories:

### Core Identifiers
- `submission_date`: The date when pings were received (partition key)
- `client_id`: Unique identifier for the Firefox client
- `sample_id`: 0-99 sampling bucket derived from client_id
- `profile_group_id`: UUID identifying the profile group
- `document_id`: First document ID from the day's pings

### Client Characteristics
- **System Information**: OS, CPU, memory, graphics features
- **Build Information**: App version, build ID, update channel
- **Geographic Data**: Country, city, subdivisions, ISP information
- **Partner Information**: Distribution ID, partner ID, attribution data

### Engagement Metrics
- **Activity**: Active hours, URI counts, search counts
- **Sessions**: Session start counts, subsession hours
- **Browser UI**: Tab/window counts, toolbar interactions

### Feature Usage
- **Add-ons**: Active add-on details and counts
- **Search**: Search counts by engine and source
- **Developer Tools**: Usage counts for various devtools panels
- **Contextual Services**: Quick Suggest impressions, clicks, topsites interactions

### Stability Metrics
- **Crashes**: Crash counts by process type (main, content, plugin)
- **Hangs**: Plugin hang occurrences
- **Shutdown**: Shutdown kill events

### Configuration
- **Search Engine**: Default search engine details
- **Preferences**: Selected user preferences (e.g., search suggestions, Quick Suggest settings)
- **Experiments**: Active experiments and branches
- **Services**: Sync configuration, Firefox Account status

## Usage Examples

### Basic Client Activity Analysis

```sql
SELECT
  submission_date,
  COUNT(DISTINCT client_id) AS dau,
  AVG(active_hours_sum) AS avg_active_hours,
  AVG(scalar_parent_browser_engagement_total_uri_count_sum) AS avg_uri_count
FROM
  `moz-fx-data-shared-prod.telemetry.clients_daily`
WHERE
  submission_date = '2024-01-15'
  AND normalized_channel = 'release'
GROUP BY
  submission_date
```

### Sampling for Performance

```sql
-- Use 1% sample for faster queries
SELECT
  normalized_channel,
  COUNT(DISTINCT client_id) AS clients
FROM
  `moz-fx-data-shared-prod.telemetry.clients_daily`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
  AND sample_id = 0  -- 1% sample
GROUP BY
  normalized_channel
```

### Add-on Analysis

```sql
SELECT
  addon.addon_id,
  addon.name,
  COUNT(DISTINCT client_id) AS client_count
FROM
  `moz-fx-data-shared-prod.telemetry.clients_daily`,
  UNNEST(active_addons) AS addon
WHERE
  submission_date = '2024-01-15'
  AND normalized_channel = 'release'
  AND addon.addon_id IS NOT NULL
GROUP BY
  addon.addon_id,
  addon.name
ORDER BY
  client_count DESC
LIMIT 20
```

### Search Behavior Analysis

```sql
SELECT
  sc.engine,
  sc.source,
  SUM(sc.count) AS total_searches
FROM
  `moz-fx-data-shared-prod.telemetry.clients_daily`,
  UNNEST(search_counts) AS sc
WHERE
  submission_date = '2024-01-15'
  AND normalized_channel = 'release'
GROUP BY
  sc.engine,
  sc.source
ORDER BY
  total_searches DESC
```

### Crash Rate Analysis

```sql
SELECT
  submission_date,
  normalized_channel,
  COUNTIF(crash_submit_success_main_sum > 0) AS clients_with_crashes,
  COUNT(*) AS total_clients,
  SAFE_DIVIDE(
    COUNTIF(crash_submit_success_main_sum > 0),
    COUNT(*)
  ) AS crash_rate
FROM
  `moz-fx-data-shared-prod.telemetry.clients_daily`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY
  submission_date,
  normalized_channel
ORDER BY
  submission_date,
  normalized_channel
```

## Related Tables

### Upstream Tables
- **`telemetry_stable.main_v5`**: Raw main ping data source
- **`telemetry.main`**: Stable main ping view

### Downstream Tables
- **`telemetry_derived.clients_last_seen_v1`**: Rolling window of client activity
- **`telemetry_derived.clients_last_seen_joined_v1`**: Includes event ping data (referenced by public view)
- **`telemetry.clients_daily`**: Public user-facing view

### Related Tables
- **`telemetry_derived.main_summary_v4`**: Legacy predecessor table (deprecated)
- **`telemetry.events`**: Event-level telemetry data
- **`telemetry_derived.clients_first_seen_v1`**: Client acquisition and first-seen dates

## Notes

### Historical Context
- This table replaced `main_summary_v4` in November 2019
- Query dependencies were added on 2019-11-22 for specific schema fields
- For data before 2019-11-22, see [legacy query version](https://github.com/mozilla/bigquery-etl/blob/813a485/sql/moz-fx-data-shared-prod/telemetry_derived/clients_daily_v6/query.sql)

### Query Best Practices
1. **Always filter by `submission_date`**: Partition filtering is required and dramatically improves query performance
2. **Use `sample_id` for exploration**: Use `sample_id = 0` for 1% sample, or `sample_id < 10` for 10% sample
3. **Check for NULL values**: Many metrics can be NULL; use `COALESCE()` or `IFNULL()` as needed
4. **Understand aggregation logic**: Configuration fields use "mode_last" (most common value from most recent pings)
5. **Be aware of data freshness**: Data is available ~2-3 hours after the submission date UTC boundary

### Known Limitations
- Does not include mobile Firefox data (use `org_mozilla_firefox.baseline_clients_daily` for mobile)
- Excludes overactive clients to prevent data quality issues
- Some fields are deprecated and return NULL (e.g., `active_experiment_id`)
- Geographic data depends on IP geolocation accuracy
- Attribution data may be missing for clients without attribution information

### Data Privacy
- All data is aggregated and anonymized according to Mozilla's [data collection policies](https://www.mozilla.org/privacy/firefox/)
- Contains only telemetry from users who have not opted out
- Does not include identifiable personal information

## Support and Contact

- **Owner**: ascholtz@mozilla.com
- **Application**: Firefox Desktop
- **GitHub Issues**: [bigquery-etl repository](https://github.com/mozilla/bigquery-etl/issues)
- **Documentation**: [Firefox Data Documentation](https://docs.telemetry.mozilla.org/)

---

**Last Updated**: 2024
**Table Version**: v6
