# Clients Daily (`clients_daily_v6`)

## Overview

The `clients_daily_v6` table provides a **daily aggregation of Firefox desktop client telemetry** data, partitioned by day. This table is Mozilla's primary source for understanding daily Firefox desktop user behavior, browser health, and feature engagement metrics. It consolidates data from thousands of individual telemetry pings per client into a single daily record per client, making it the go-to table for most desktop Firefox analysis.

### Key Characteristics

- **Granularity**: One row per client per day
- **Source**: `telemetry_stable.main_v5` (raw main pings)
- **Data Platform**: BigQuery
- **Partition Key**: `submission_date` (DATE)
- **Clustering**: `normalized_channel`, `sample_id`
- **Retention**: 775 days (~2.1 years)
- **Update Frequency**: Daily
- **DAG**: `bqetl_main_summary`
- **Owner**: ascholtz@mozilla.com

### Table Purpose

This table serves multiple critical purposes:

1. **User Behavior Analysis**: Track browsing patterns, search activity, tab/window usage
2. **Product Health Monitoring**: Monitor crashes, stability, performance metrics
3. **Feature Adoption**: Measure usage of Firefox features (DevTools, Sync, add-ons, etc.)
4. **Search & Revenue**: Analyze search behavior, ad impressions, and monetization
5. **Platform Intelligence**: Understand OS distribution, hardware capabilities, geo distribution
6. **Experiment Analysis**: Support A/B testing and feature rollout evaluation

### Data Freshness

- **Processing Time**: Typically available by 8 AM UTC the following day
- **Lookback Window**: Data for date `D` represents activity from `D 00:00` to `D 23:59` UTC (based on `submission_timestamp`)
- **Downstream Dependencies**: Feeds into `clients_last_seen`, operational monitoring, Jetstream, and parquet exports

## Schema

The table contains **over 400 columns** organized into these categories:

### Core Dimensions
- **Identity**: `client_id`, `sample_id`, `submission_date`
- **Geographic**: `country`, `city`, `geo_subdivision1`, `geo_subdivision2`, `isp_name`
- **Platform**: `os`, `os_version`, `normalized_os_version`, `normalized_channel`
- **Application**: `app_version`, `app_display_version`, `app_build_id`, `channel`

### Engagement Metrics
- **Activity**: `active_hours_sum`, `subsession_hours_sum`, `pings_aggregated_by_this_row`
- **Browser Use**: `scalar_parent_browser_engagement_total_uri_count_sum`, `scalar_parent_browser_engagement_max_concurrent_tab_count_max`
- **Sessions**: `sessions_started_on_this_day`, `session_restored_mean`

### Search Metrics
- **Search Counts**: `search_counts` (structured by engine × source), `search_count_all`
- **Monetization**: `search_with_ads_count_all`, `ad_clicks_count_all`
- **Access Points**: `search_count_urlbar`, `search_count_searchbar`, `search_count_abouthome`, etc.
- **Detailed Breakdown**: `search_content_urlbar_sum`, `search_withads_urlbar_sum`, `search_adclicks_urlbar_sum` (keyed by engine)

### Stability & Performance
- **Crashes**: `crashes_detected_content_sum`, `crash_submit_success_main_sum`, etc.
- **Hangs**: `plugin_hangs_sum`, `shutdown_kill_sum`
- **Latency**: `first_paint_mean`, `client_submission_latency_mean`

### Feature Usage
- **DevTools**: ~40 columns tracking developer tool usage (e.g., `histogram_parent_devtools_webconsole_opened_count_sum`)
- **Add-ons**: `active_addons`, `active_addons_count_mean`
- **Sync**: `sync_configured`, `sync_count_desktop_sum`, `sync_count_mobile_sum`, `fxa_configured`
- **Firefox Suggest**: ~50 columns tracking Quick Suggest interactions
- **Accessibility**: `a11y_theme`, `scalar_a11y_hcm_foreground`, accessibility-related metrics

### Hardware & Environment
- **CPU**: `cpu_cores`, `cpu_count`, `cpu_vendor`, `cpu_speed_mhz`, `cpu_model`
- **Memory**: `memory_mb`
- **Graphics**: `gfx_features_d3d11_status`, `gfx_features_gpu_process_status`
- **System**: `is_wow64`, `windows_build_number`, `apple_model_id`

### User Preferences
- **Search Settings**: `user_pref_browser_search_suggest_enabled`, `user_pref_browser_urlbar_suggest_searches`
- **Firefox Suggest**: `user_pref_browser_urlbar_suggest_quicksuggest_sponsored`
- **Privacy**: `user_pref_browser_newtabpage_enabled`, `user_pref_app_shield_optoutstudies_enabled`

### Migration & Onboarding
- **Browser Migrations**: `bookmark_migrations_quantity_chrome`, `history_migrations_quantity_safari`, `logins_migrations_quantity_edge`
- **Quantity Totals**: `bookmark_migrations_quantity_all`, `history_migrations_quantity_all`

### Complex Nested Structures
Many columns are **REPEATED RECORD types** (arrays of key-value pairs):
- Search metrics by engine: `search_counts`, `search_with_ads`, `ad_clicks`
- URL bar interactions: `scalar_parent_urlbar_picked_*_sum`, `scalar_parent_urlbar_searchmode_*_sum`
- Firefox Suggest: `contextual_services_quicksuggest_*_sum`
- Telemetry events: `scalar_parent_telemetry_event_counts_sum`

## Data Sources

### Primary Source
- **Table**: `telemetry_stable.main_v5`
- **Ping Type**: `main` pings from Firefox desktop
- **Submission**: Sent periodically (typically on browser shutdown, daily subsession boundary, or environment change)

### ETL Pipeline Overview
The query performs complex aggregation through multiple CTEs:

1. **Base CTE**: Reads from `main_v5`, extracts histogram data using UDFs
2. **Overactive Filter**: Excludes clients with >150,000 pings/day (data quality filter)
3. **Clients Summary**: Extracts and transforms raw ping fields per ping
4. **Aggregates**: Aggregates to client-day level using SUM, AVG, MAX, mode_last
5. **UDF Aggregates**: Further aggregates keyed scalars and histograms into map structures

### Aggregation Logic
- **Numeric metrics**: Typically SUM for counts, AVG for rates, MAX for peak values
- **Categorical fields**: `mozfun.stats.mode_last()` (most recent value)
- **Nested structures**: `mozfun.map.sum()` for key-value aggregation
- **Time conversion**: Active hours = `active_ticks / (3600/5)` (5-second tick intervals)

## Update Schedule

### Scheduling Details
- **DAG**: `bqetl_main_summary`
- **Frequency**: Daily
- **Start Date**: 2019-11-05
- **Typical Run Time**: ~2-3 hours after midnight UTC
- **Partition Filter**: Required (queries must filter on `submission_date`)

### Downstream Dependencies
This table has multiple downstream consumers that wait for completion:

| Consumer | DAG | Wait Time |
|----------|-----|-----------|
| Jetstream (experiment analysis) | `jetstream` | +2 hours |
| Operational Monitoring | `operational_monitoring` | +2 hours |
| Parquet Export | `parquet_export` | +1 hour |

### Backfill Considerations
- **Historical Data**: Available from 2019-11-05 onward
- **Schema Evolution**: For backfills before 2019-11-22, use historical query version (referenced in query comments)
- **Partition Expiration**: Data >775 days is automatically deleted

## Usage Examples

### Example 1: Daily Active Users by Channel

```sql
SELECT
  submission_date,
  normalized_channel,
  COUNT(DISTINCT client_id) AS dau
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 28 DAY)
GROUP BY
  submission_date,
  normalized_channel
ORDER BY
  submission_date DESC,
  dau DESC
```

### Example 2: Search Volume by Engine and Source

```sql
SELECT
  submission_date,
  search.engine,
  search.source,
  SUM(search.count) AS total_searches
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`,
  UNNEST(search_counts) AS search
WHERE
  submission_date = '2024-01-15'
  AND search.engine IN ('google', 'bing', 'ddg')
GROUP BY
  submission_date,
  search.engine,
  search.source
ORDER BY
  total_searches DESC
```

### Example 3: Crash Rate by OS

```sql
SELECT
  os,
  COUNT(DISTINCT client_id) AS total_clients,
  COUNTIF(crash_submit_success_main_sum > 0) AS clients_with_crashes,
  SAFE_DIVIDE(
    COUNTIF(crash_submit_success_main_sum > 0),
    COUNT(DISTINCT client_id)
  ) AS crash_rate,
  AVG(crash_submit_success_main_sum) AS avg_crashes_per_client
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date = CURRENT_DATE() - 1
GROUP BY
  os
HAVING
  total_clients > 1000
ORDER BY
  crash_rate DESC
```

### Example 4: Firefox Suggest Engagement

```sql
SELECT
  submission_date,
  COUNT(DISTINCT client_id) AS clients_with_impressions,
  SUM((SELECT SUM(value) FROM UNNEST(contextual_services_quicksuggest_impression_sum))) AS total_impressions,
  SUM((SELECT SUM(value) FROM UNNEST(contextual_services_quicksuggest_click_sum))) AS total_clicks,
  SAFE_DIVIDE(
    SUM((SELECT SUM(value) FROM UNNEST(contextual_services_quicksuggest_click_sum))),
    SUM((SELECT SUM(value) FROM UNNEST(contextual_services_quicksuggest_impression_sum)))
  ) AS click_through_rate
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
  AND normalized_channel = 'release'
GROUP BY
  submission_date
ORDER BY
  submission_date
```

### Example 5: Hardware Distribution

```sql
SELECT
  cpu_vendor,
  memory_mb,
  os,
  COUNT(DISTINCT client_id) AS client_count
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date = CURRENT_DATE() - 1
  AND cpu_vendor IS NOT NULL
  AND memory_mb IS NOT NULL
GROUP BY
  cpu_vendor,
  memory_mb,
  os
ORDER BY
  client_count DESC
LIMIT 20
```

### Example 6: Add-on Usage Analysis

```sql
WITH addon_data AS (
  SELECT
    submission_date,
    client_id,
    addon.addon_id,
    addon.name
  FROM
    `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`,
    UNNEST(active_addons) AS addon
  WHERE
    submission_date = CURRENT_DATE() - 1
    AND addon.addon_id IS NOT NULL
)
SELECT
  addon_id,
  ANY_VALUE(name) AS addon_name,
  COUNT(DISTINCT client_id) AS users
FROM
  addon_data
GROUP BY
  addon_id
ORDER BY
  users DESC
LIMIT 100
```

### Example 7: Sampling Pattern (for Large Queries)

```sql
-- Use sample_id for 1% sample (sample_id = 0 to 99)
SELECT
  submission_date,
  COUNT(DISTINCT client_id) AS sampled_dau,
  COUNT(DISTINCT client_id) * 100 AS estimated_total_dau
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date >= '2024-01-01'
  AND sample_id = 0  -- 1% sample
GROUP BY
  submission_date
ORDER BY
  submission_date DESC
```

## Related Tables

### Upstream Dependencies
- **`telemetry_stable.main_v5`**: Raw main ping data (source table)

### Downstream Tables
- **`telemetry.clients_daily`**: User-facing view with stable schema
- **`telemetry_derived.clients_last_seen_joined_v1`**: Extended with event ping data
- **`telemetry_derived.clients_last_seen_v1`**: Longitudinal view with retention metrics
- **`telemetry.main_summary`**: Historical table (deprecated, use clients_daily instead)

### Related Desktop Tables
- **`telemetry_derived.main_summary_v4`**: Predecessor table (deprecated)
- **`telemetry_derived.core_clients_daily_v1`**: Similar structure for Firefox Android
- **`telemetry.crash`**: Detailed crash reports
- **`telemetry.event`**: Granular event-level data

### Search-Specific Tables
- **`search_derived.search_clients_daily_v8`**: More detailed search metrics
- **`search_derived.search_rfm_v1`**: Search RFM (Recency, Frequency, Monetary) analysis

## Notes

### Important Considerations

1. **Always Filter on `submission_date`**: This table is partitioned and requires partition filtering for efficient queries. Queries without date filters will fail.

2. **Sample for Large Queries**: Use `sample_id` (0-99) for 1% or 10% samples when exploring data or running expensive queries.

3. **NULL Descriptions**: Many columns have NULL descriptions in the schema. Enhanced descriptions are now available in this documentation.

4. **Nested Structures**: Many metrics use REPEATED RECORD types. Use `UNNEST()` to flatten arrays when querying.

5. **Mode vs. Sum**:
   - Most categorical fields use `mozfun.stats.mode_last()` (most recently seen value)
   - Numeric counts use `SUM()` aggregation
   - Peak values use `MAX()`

6. **Deprecated Fields**: Several fields return NULL (e.g., `active_experiment_id`, `total_hours_sum`, `histogram_parent_devtools_developertoolbar_opened_count_sum`)

7. **Data Quality**:
   - Excludes "overactive" clients with >150,000 pings/day
   - Filters out non-Firefox applications (`normalized_app_name = 'Firefox'`)
   - Requires non-null `document_id`

8. **Attribution Data**: Marketing attribution is available in the `attribution` struct but may be NULL for many clients

9. **Search Metrics Duplication**: Search data exists in multiple formats:
   - Aggregated: `search_count_all`, `search_count_urlbar`, etc.
   - Detailed by engine: `search_counts` (REPEATED RECORD)
   - Detailed by engine × source: `search_content_urlbar_sum`, etc. (REPEATED RECORD)

   Choose the appropriate level based on your analysis needs.

10. **Firefox Suggest Metrics**: Extensive Quick Suggest telemetry is available with granular tracking of:
    - Impressions, clicks, blocks, and help interactions
    - Segmented by type: sponsored, non-sponsored, best match, weather, Wikipedia
    - Position-level tracking in nested structures

### Schema Evolution

- **2019-11-22**: Major schema update with added fields (active_addons details, paint timing)
- **2020-2021**: Addition of Firefox Suggest (Quick Suggest) metrics
- **2021-2022**: Expanded search telemetry with access point breakdown
- **2023**: Added sidebar, library, and enhanced accessibility metrics

For historical queries before these dates, consult the query.sql comment header for backfill guidance.

### Performance Tips

1. **Use the user-facing view** `telemetry.clients_daily` when possible (cleaner schema, better documentation)
2. **Leverage clustering**: Filter on `normalized_channel` and `sample_id` early in queries
3. **Avoid `SELECT *`**: This table has 400+ columns; specify only needed fields
4. **Use date ranges wisely**: Limit to necessary date ranges to minimize data scanned
5. **Pre-aggregate when possible**: Create intermediate tables for complex repeated analyses

### Getting Help

- **Data Documentation**: https://docs.telemetry.mozilla.org/
- **Code Repository**: https://github.com/mozilla/bigquery-etl
- **#data-help**: Mozilla's internal Slack channel for data questions
- **Owner**: ascholtz@mozilla.com

---

**Table**: `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
**Last Updated**: 2024
**Schema Version**: v6
