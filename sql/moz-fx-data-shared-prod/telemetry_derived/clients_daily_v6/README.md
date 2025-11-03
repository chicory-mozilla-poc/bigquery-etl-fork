# clients_daily_v6

## Overview

**Table**: `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`

The `clients_daily_v6` table provides a comprehensive daily aggregation of Firefox desktop telemetry data at the client level. This table consolidates multiple `main` pings received from each Firefox desktop client within a single day, creating a unified view of client activity, system configuration, and user behavior.

This derived table serves as a fundamental building block for Firefox desktop analytics and should typically be accessed through the user-facing view `telemetry.clients_daily`. As of Q1 2021, this view references the downstream table `clients_last_seen_joined_v1`, which merges additional fields from the `event` ping.

## Purpose

- **Client-Level Aggregation**: Consolidates multiple daily pings per client into a single row per client per day
- **Performance Optimization**: Pre-aggregates metrics to reduce query complexity and improve performance
- **Comprehensive Coverage**: Includes system configuration, user engagement, search behavior, crash data, and developer tool usage
- **Foundation for Analysis**: Powers downstream tables, dashboards, and analytical workloads across Mozilla

## Data Sources

### Primary Source
- **Table**: `moz-fx-data-shared-prod.telemetry_stable.main_v5`
- **Ping Type**: Firefox Desktop `main` ping
- **Filter**: `normalized_app_name = 'Firefox'` and valid `document_id`

### Data Quality Controls
- **Overactive Client Filtering**: Excludes clients with >150,000 pings/day or >3,000,000 active addons (prevents aggregation overflows)
- **Date-Based Partitioning**: Processes data based on `submission_timestamp` date

## Schema

The table contains **363 columns** organized into the following categories:

### Core Identifiers
- `submission_date` (DATE): Server-side date when telemetry was received
- `client_id` (STRING): Unique client identifier (UUID)
- `sample_id` (INTEGER): Sampling bucket (0-99) for data filtering
- `document_id` (STRING): First document ID from the day's pings

### System Configuration
- **Operating System**: `os`, `os_version`, `normalized_os_version`, `windows_build_number`, `windows_ubr`
- **Hardware**: `cpu_*` fields (vendor, model, cores, speed), `memory_mb`, `apple_model_id`
- **Graphics**: `gfx_features_*` fields (D3D11, D2D, GPU process status)

### Application Information
- **Build Details**: `app_build_id`, `app_version`, `app_display_version`, `channel`, `normalized_channel`
- **Distribution**: `distribution_id`, `partner_id`, `distributor`, `distributor_channel`
- **Environment**: `env_build_*` fields, `locale`, `timezone_offset`

### User Engagement Metrics
- **Activity**: `active_hours_sum`, `subsession_hours_sum`, `sessions_started_on_this_day`
- **Browser Usage**: `scalar_parent_browser_engagement_total_uri_count_sum`, `scalar_parent_browser_engagement_unique_domains_count_*`
- **Tabs and Windows**: `scalar_parent_browser_engagement_max_concurrent_tab_count_max`, `scalar_parent_browser_engagement_max_concurrent_window_count_max`

### Search Behavior
- **Search Counts**: `search_counts` (array with engine and source breakdown)
- **Access Points**: `search_count_*` fields (urlbar, searchbar, about:home, newtab, contextmenu)
- **Ad Engagement**: `ad_clicks_count_all`, `search_with_ads_count_all`
- **Search Preferences**: `user_pref_browser_search_*` fields

### Addons and Extensions
- `active_addons` (REPEATED RECORD): List of active addons with metadata
- `active_addons_count_mean`: Average number of active addons

### Crash and Stability
- **Crashes**: `crashes_detected_*_sum` (content, plugin, gmplugin)
- **Crash Reporting**: `crash_submit_attempt_*_sum`, `crash_submit_success_*_sum`
- **Aborts**: `aborts_*_sum` (content, plugin, gmplugin)
- **Hangs**: `plugin_hangs_sum`, `shutdown_kill_sum`

### Developer Tools
- **Histogram Counts**: `histogram_parent_devtools_*_opened_count_sum` (inspector, console, debugger, etc.)
- **Scalars**: `scalar_parent_devtools_*` (accessibility, copy selectors, eyedropper)

### Privacy and Security
- **Tracking Protection**: `trackers_blocked_sum`
- **SSL**: `ssl_handshake_result_success_sum`, `ssl_handshake_result_failure_sum`
- **Sandbox**: `sandbox_effective_content_process_level`

### Sync and Services
- **Firefox Account**: `fxa_configured`, `sync_configured`
- **Sync Devices**: `sync_count_desktop_*`, `sync_count_mobile_*`

### Geographic and Network
- **Location**: `country`, `city`, `geo_subdivision1`, `geo_subdivision2`
- **ISP**: `isp_name`, `isp_organization`

### Contextual Services
- **Quick Suggest**: `contextual_services_quicksuggest_*` (impressions, clicks, blocks)
- **Top Sites**: `contextual_services_topsites_*`

### URL Bar Interactions
- **Search Modes**: `scalar_parent_urlbar_searchmode_*`
- **Picked Results**: `scalar_parent_urlbar_picked_*` (autofill, bookmark, history, search, etc.)
- **Impressions**: `scalar_parent_urlbar_impression_autofill_*`

### User Preferences
- Multiple `user_pref_*` fields capturing browser configuration preferences

## Update Schedule

- **Frequency**: Daily
- **DAG**: `bqetl_main_summary`
- **Start Date**: 2019-11-05
- **Partition**: By `submission_date` (day-level partitioning)
- **Partition Filter**: Required (queries must include `submission_date` filter)
- **Data Retention**: 775 days (~2 years)

## Downstream Consumers

This table serves as a critical dependency for:
- **Jetstream**: Experiment analysis platform (wait_for_clients_daily, 2h execution delta)
- **Operational Monitoring**: Real-time monitoring dashboards (wait_for_clients_daily, 2h execution delta)
- **Parquet Export**: Data export pipeline (wait_for_clients_daily, 1h execution delta)
- **clients_last_seen**: Longitudinal client history table
- **Various Dashboards**: GUD (Growth and Usage Dashboard), user retention analyses

## Usage Examples

### Example 1: Daily Active Users (DAU) by Channel
```sql
SELECT
  submission_date,
  normalized_channel,
  COUNT(DISTINCT client_id) AS dau
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date = '2024-01-15'
GROUP BY
  submission_date,
  normalized_channel
ORDER BY
  normalized_channel
```

### Example 2: Average Active Hours by Country
```sql
SELECT
  country,
  AVG(active_hours_sum) AS avg_active_hours,
  COUNT(DISTINCT client_id) AS client_count
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
  AND country IS NOT NULL
GROUP BY
  country
HAVING
  client_count >= 1000
ORDER BY
  avg_active_hours DESC
LIMIT 20
```

### Example 3: Search Behavior Analysis
```sql
SELECT
  submission_date,
  search_count.engine,
  search_count.source,
  SUM(search_count.count) AS total_searches
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`,
  UNNEST(search_counts) AS search_count
WHERE
  submission_date = '2024-01-15'
  AND search_count.engine IS NOT NULL
GROUP BY
  submission_date,
  search_count.engine,
  search_count.source
ORDER BY
  total_searches DESC
LIMIT 20
```

### Example 4: Crash Rate Analysis
```sql
SELECT
  submission_date,
  normalized_channel,
  COUNT(DISTINCT client_id) AS total_clients,
  COUNTIF(crashes_detected_content_sum > 0) AS clients_with_crashes,
  SAFE_DIVIDE(
    COUNTIF(crashes_detected_content_sum > 0),
    COUNT(DISTINCT client_id)
  ) * 100 AS crash_rate_pct
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY
  submission_date,
  normalized_channel
ORDER BY
  submission_date,
  normalized_channel
```

### Example 5: Top Active Addons
```sql
SELECT
  addon.addon_id,
  addon.name,
  COUNT(DISTINCT client_id) AS client_count
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`,
  UNNEST(active_addons) AS addon
WHERE
  submission_date = '2024-01-15'
  AND addon.addon_id IS NOT NULL
  AND addon.is_system = FALSE
GROUP BY
  addon.addon_id,
  addon.name
ORDER BY
  client_count DESC
LIMIT 20
```

## Related Tables

- **telemetry.clients_daily**: User-facing view (recommended access point)
- **telemetry_derived.clients_last_seen_v1**: Longitudinal client history (28-day retention)
- **telemetry_derived.clients_last_seen_joined_v1**: Merged with event ping data
- **telemetry_stable.main_v5**: Source table (raw main pings)
- **telemetry_derived.main_summary_v4**: Legacy summary table (deprecated)

## Important Notes

### Data Quality Considerations
1. **Overactive Client Filtering**: The query excludes clients with excessive ping volume to prevent aggregation errors
2. **NULL Descriptions**: Some columns in the schema have NULL descriptions; refer to this README and the source query for context
3. **Aggregation Functions**: The table uses various aggregation strategies:
   - `mode_last`: Takes the most recent mode (most common value with last-seen tiebreaker)
   - `SUM`: Aggregates numeric values across all pings
   - `AVG/MEAN`: Computes averages across pings
   - `MAX`: Takes maximum value observed

### Schema Evolution
- The table structure has evolved over time; new fields are added as Firefox telemetry expands
- Deprecated fields (e.g., `active_experiment_id`, `total_hours_sum`) remain in schema for backwards compatibility but return NULL
- Always reference the latest query.sql for authoritative field definitions

### Performance Tips
1. **Always filter on `submission_date`**: Partition filtering is required and significantly improves query performance
2. **Use `sample_id` for sampling**: For exploratory analysis, filter to `sample_id = 0` for a 1% sample
3. **Leverage clustering**: The table is clustered by `normalized_channel` and `sample_id`
4. **Prefer aggregated metrics**: This table pre-aggregates data; avoid re-aggregating when possible

### Access and Permissions
- **Owner**: ascholtz@mozilla.com
- **Application**: Firefox Desktop
- **Table Type**: Client-level aggregation
- **Schedule**: Daily via Airflow DAG `bqetl_main_summary`

## References

- [BigQuery ETL Repository](https://github.com/mozilla/bigquery-etl)
- [Issue #1761: clients_last_seen_joined_v1 migration](https://github.com/mozilla/bigquery-etl/issues/1761)
- [Main Ping Documentation](https://firefox-source-docs.mozilla.org/toolkit/components/telemetry/data/main-ping.html)
- [Telemetry Data Platform Documentation](https://docs.telemetry.mozilla.org/)

## Version History

- **v6**: Current version with enhanced metrics and improved aggregations
- **v5**: Previous version (deprecated)
- **v4 and earlier**: Legacy versions (main_summary_v4 table)

---

**Last Updated**: 2025-01-19 (Auto-generated documentation)
**Table Owner**: ascholtz@mozilla.com
