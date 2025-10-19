# Clients Daily v6

## Overview

The `clients_daily_v6` table provides a comprehensive daily aggregated view of Firefox desktop client telemetry data. This table serves as the primary data source for understanding Firefox client behavior, system configuration, and usage patterns on a per-client, per-day basis.

This table aggregates multiple telemetry pings (from `telemetry_stable.main_v5`) received from each Firefox client per day into a single row, providing a holistic daily snapshot of client activity, system characteristics, and behavioral metrics.

## Table-Level Quality Description

**Purpose**: Provides daily aggregated Firefox desktop telemetry metrics for analysis of client behavior, browser performance, system characteristics, user engagement, search activity, and feature usage.

**Data Sources**:
- Primary source: `moz-fx-data-shared-prod.telemetry_stable.main_v5` (Firefox main ping data)
- Depends on Firefox clients sending main pings with telemetry data

**Data Grain**: One row per `client_id` per `submission_date`

**Update Schedule**: Daily incremental updates, partitioned by `submission_date`

**Key Characteristics**:
- Aggregates multiple pings per client per day using various aggregation methods (SUM for counters, AVG for measurements, mode_last for configuration values)
- Filters out "overactive" clients (>150,000 pings/day or >3M active addons) to prevent aggregation errors
- Includes comprehensive metrics across dimensions: usage, performance, system info, search behavior, addons, experiments, and developer tools
- Supports cohort analysis, A/B testing evaluation, and longitudinal user behavior studies

**Data Quality Notes**:
- Telemetry data is subject to client-side collection and network availability
- Geographic and ISP information derived from IP geolocation at ingestion time
- Some fields use `mode_last` aggregation to capture the most recent state within a day
- Historical data prior to 2019-11-22 requires special backfill handling (see query comments)

## Schema

For detailed column-level documentation, please refer to the schema.yaml file in this directory. The table contains 300+ columns organized into the following categories:

- **Core Identifiers**: submission_date, client_id, sample_id, document_id, profile_group_id
- **Aggregation Metadata**: pings_aggregated_by_this_row, submission_timestamp_min, subsession_hours_sum, active_hours_sum, sessions_started_on_this_day
- **Application & Build Info**: app_build_id, app_display_version, app_name, app_version, channel, normalized_channel
- **Operating System & Hardware**: os, os_version, memory_mb, cpu_count, cpu_cores, cpu_vendor, cpu_model
- **Geographic & Network**: country, city, geo_subdivision1, geo_subdivision2, isp_name
- **User Engagement**: total_uri_count, unique_domains_count, tab_open_event_count, window_open_event_count
- **Search Activity**: search_count_all, search_with_ads_count_all, ad_clicks_count_all, search_counts, default_search_engine
- **Add-ons & Extensions**: active_addons, active_addons_count_mean
- **Sync & FxA**: sync_configured, sync_count_desktop, sync_count_mobile, fxa_configured
- **Experiments**: experiments array with experiment names and branches
- **Crashes & Stability**: crashes_detected_*, crash_submit_*, aborts_*
- **Security & Privacy**: ssl_handshake_result_*, trackers_blocked_sum, sandbox_effective_content_process_level
- **Browser History & Bookmarks**: places_bookmarks_count_mean, places_pages_count_mean
- **Developer Tools**: devtools_toolbox_opened_count_sum, histogram_parent_devtools_*_opened_count_sum
- **And many more...**

## Data Sources

### Primary Source Table
- **`moz-fx-data-shared-prod.telemetry_stable.main_v5`**
  - Contains raw Firefox telemetry main ping data
  - Filtered to `normalized_app_name = 'Firefox'` and `document_id IS NOT NULL`
  - Data is partitioned by `DATE(submission_timestamp)`

### Data Quality Filters
- **Overactive Client Filtering**: Excludes clients with:
  - More than 150,000 pings in a single day
  - More than 3,000,000 total active addons across all pings
  - These thresholds prevent aggregation overflow errors

## Update Schedule

- **Frequency**: Daily
- **Partition**: `submission_date` (DATE)
- **Backfill Consideration**: For dates prior to 2019-11-22, use the historical version of this query that reads from `main_summary_v4`

## Usage Examples

### Daily Active Users by Channel
```sql
SELECT
  submission_date,
  normalized_channel,
  COUNT(DISTINCT client_id) AS dau
FROM `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE submission_date = CURRENT_DATE() - 1
GROUP BY submission_date, normalized_channel
```

### Average Session Time by Country
```sql
SELECT
  country,
  AVG(active_hours_sum) AS avg_active_hours,
  COUNT(DISTINCT client_id) AS user_count
FROM `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE submission_date BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY) AND CURRENT_DATE()
  AND country IS NOT NULL
GROUP BY country
HAVING user_count > 1000
ORDER BY user_count DESC
LIMIT 20
```

### Search Engagement Analysis
```sql
SELECT
  submission_date,
  SUM(search_count_all) AS total_searches,
  SUM(search_with_ads_count_all) AS searches_with_ads,
  SUM(ad_clicks_count_all) AS ad_clicks,
  SAFE_DIVIDE(SUM(ad_clicks_count_all), SUM(search_with_ads_count_all)) AS ctr
FROM `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY submission_date
ORDER BY submission_date
```

### Cohort Retention Analysis
```sql
WITH cohorts AS (
  SELECT
    client_id,
    MIN(submission_date) AS first_seen_date
  FROM `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
  WHERE submission_date >= '2024-01-01'
  GROUP BY client_id
)
SELECT
  cohorts.first_seen_date AS cohort_date,
  DATE_DIFF(cd.submission_date, cohorts.first_seen_date, DAY) AS days_since_first_seen,
  COUNT(DISTINCT cd.client_id) AS active_users
FROM cohorts
INNER JOIN `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6` cd
  ON cohorts.client_id = cd.client_id
WHERE cd.submission_date BETWEEN cohorts.first_seen_date AND DATE_ADD(cohorts.first_seen_date, INTERVAL 30 DAY)
GROUP BY cohort_date, days_since_first_seen
ORDER BY cohort_date, days_since_first_seen
```

## Related Tables

- **`telemetry.main`**: Raw, unnested main ping data (more granular than clients_daily)
- **`telemetry_derived.main_summary_v4`**: Predecessor table (deprecated, use clients_daily_v6)
- **`telemetry_derived.clients_last_seen_v1`**: Complementary table tracking client activity windows
- **`telemetry_derived.core_clients_daily_v1`**: Simplified subset of core metrics
- **`telemetry.new_profile`**: New profile creation events
- **`telemetry.first_shutdown`**: First shutdown ping data

## Notes

1. **Aggregation Methods**: Different fields use different aggregation strategies:
   - `mode_last`: Takes the last seen value (most recent ping)
   - `SUM`: Adds up values across all pings
   - `AVG`: Averages values across pings
   - `MAX`/`MIN`: Takes maximum/minimum values
   - Custom UDFs: Used for complex types like addons and search counts

2. **Sample ID Usage**: For performance in large queries, use `sample_id` to analyze subsets:
   - `sample_id = 42` gives ~1% of users (deterministic)
   - `sample_id < 10` gives ~10% of users

3. **Geographic Data**: Country and city are determined via IP geolocation at ingestion time and may not reflect VPN or proxy usage.

4. **Time Zones**: All timestamps are in UTC. Use `timezone_offset` for client local time conversions.

5. **Null Values**: Many fields can be NULL due to:
   - Feature not being used (e.g., no searches = NULL search counts)
   - Data not collected in older Firefox versions
   - Client-side collection errors or privacy settings

6. **Private Browsing**: Most metrics do not include private browsing activity except where explicitly noted.

7. **Performance**: This table is large (~millions of rows per day). Always:
   - Filter by `submission_date` first
   - Consider using `sample_id` for analysis
   - Limit the number of columns in SELECT statements

8. **Historical Changes**: Schema has evolved over time. Some fields were added in specific Firefox versions. Check field availability when analyzing historical data.

## Contact & Support

For questions about this table:
- **Data Documentation**: [docs.telemetry.mozilla.org](https://docs.telemetry.mozilla.org)
- **Firefox Data Teams**: #data-help on Mozilla Slack
- **Bug Reports**: File in Bugzilla under Data Platform & Tools :: General

## Version History

- **v6**: Current version (query.sql in this directory)
- **v5 and earlier**: Deprecated, historical versions reading from main_summary_v4

---

🤖 Generated with enhanced documentation for `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
