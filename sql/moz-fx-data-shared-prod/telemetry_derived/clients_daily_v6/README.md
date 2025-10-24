# Clients Daily v6

## Overview

The `clients_daily_v6` table is the primary source for daily Firefox desktop telemetry analysis, providing comprehensive client-level aggregation of activity, configuration, performance, and usage metrics.

**Quick Stats:**
- **Granularity:** One row per client per day
- **Source:** `telemetry_stable.main_v5` pings
- **Partitioning:** Daily by `submission_date`
- **Retention:** 775 days (~2.1 years)
- **Update Frequency:** Daily (part of `bqetl_main_summary` DAG)
- **Clustering:** `normalized_channel`, `sample_id`

## Purpose

This table aggregates multiple main ping submissions from each Firefox desktop client into a single daily row, providing a comprehensive view of:

- **User Engagement:** Active hours, URI counts, tab/window usage
- **Search Activity:** Search counts by source, engine, and monetization status
- **Feature Usage:** DevTools, addons, sync, private browsing, and more
- **System Configuration:** Hardware specs, OS details, graphics capabilities
- **Performance & Stability:** Crashes, hangs, latencies, and resource usage
- **Experiment Participation:** A/B test assignments and Normandy studies

## Primary Use Cases

1. **Daily Active Users (DAU) Analysis**
   - Count distinct `client_id` where `submission_date = [date]`
   - Filter by `normalized_channel`, `country`, or other dimensions

2. **Feature Adoption Tracking**
   - Monitor adoption rates of new features (e.g., `e10s_enabled`, `fxa_configured`)
   - Track DevTools usage via `histogram_parent_devtools_*` columns

3. **Search Monetization**
   - Analyze search counts via `search_counts` array
   - Track ad impressions and clicks: `search_with_ads_count_all`, `ad_clicks_count_all`

4. **Hardware/OS Distribution**
   - Segment users by `os`, `cpu_vendor`, `memory_mb`
   - Analyze graphics capabilities via `gfx_features_*` columns

5. **Experiment Analysis**
   - Join on `experiments` array to analyze A/B test results
   - Compare metrics between treatment and control groups

6. **Performance Monitoring**
   - Track crash rates: `crashes_detected_*` and `crash_submit_*` columns
   - Monitor latencies: `client_submission_latency_mean`, `first_paint_mean`

## Schema

### Key Columns

#### Identity & Temporal
- `submission_date` (DATE): Server-side receipt date (partition key)
- `client_id` (STRING): Unique Firefox client identifier
- `sample_id` (INTEGER): 0-99 sampling bucket for representative analysis
- `profile_group_id` (STRING): UUID identifying profile group (not shared with other telemetry)

#### Engagement Metrics
- `active_hours_sum` (FLOAT): Total active usage hours (key DAU metric)
- `subsession_hours_sum` (NUMERIC): Total subsession duration
- `scalar_parent_browser_engagement_total_uri_count_sum` (INT): Total page loads
- `scalar_parent_browser_engagement_unique_domains_count_max` (INT): Peak unique domains visited
- `scalar_parent_browser_engagement_max_concurrent_tab_count_max` (INT): Maximum tabs open simultaneously

#### Search Metrics
- `search_counts` (ARRAY<STRUCT>): Detailed search counts by engine and source
  - `engine` (STRING): Search engine identifier
  - `source` (STRING): Search access point (urlbar, newtab, etc.)
  - `count` (INTEGER): Number of searches
- `search_with_ads_count_all` (INT): Total searches with ads shown
- `ad_clicks_count_all` (INT): Total ad clicks (revenue metric)

#### Configuration
- `normalized_channel` (STRING): Release channel (release, beta, nightly, esr)
- `app_version` (STRING): Firefox version
- `country` (STRING): ISO country code (from IP geolocation)
- `os` (STRING): Operating system (Windows_NT, Darwin, Linux)
- `locale` (STRING): UI language/locale

#### Hardware
- `memory_mb` (INT): System RAM in megabytes
- `cpu_vendor` (STRING): CPU manufacturer (GenuineIntel, AuthenticAMD)
- `cpu_cores` (INT): Physical CPU cores
- `cpu_count` (INT): Logical CPUs

#### Stability
- `crashes_detected_content_sum` (INT): Content process crashes
- `crashes_detected_plugin_sum` (INT): Plugin process crashes
- `crash_submit_success_main_sum` (INT): Successfully submitted main crash reports
- `aborts_content_sum` (INT): Content process abnormal aborts

#### Features
- `e10s_enabled` (BOOL): Multiprocess Firefox enabled
- `fxa_configured` (BOOL): Firefox Account configured
- `sync_configured` (BOOL): Sync enabled
- `is_default_browser` (BOOL): Firefox set as default browser
- `active_addons` (ARRAY): List of installed/enabled addons with metadata

### Full Schema

For the complete schema with all 500+ columns, see `schema.yaml`. Columns are organized into these categories:

1. **Temporal & Identity** (10 columns): `submission_date`, `client_id`, `sample_id`, etc.
2. **System Configuration** (50+ columns): OS, hardware, graphics, build info
3. **Engagement Metrics** (30+ columns): Active hours, URI counts, tabs, windows
4. **Search Metrics** (80+ columns): Search counts by source, engine, ads, clicks
5. **DevTools Usage** (40+ columns): Individual tool open counts
6. **Stability Metrics** (30+ columns): Crashes, hangs, aborts, shutdowns
7. **Feature Usage** (50+ columns): Addons, sync, private browsing, etc.
8. **Performance Metrics** (40+ columns): Paint times, latencies, resource usage
9. **Scalars & Histograms** (150+ columns): Various telemetry probes
10. **User Preferences** (30+ columns): Browser settings and configurations

## Data Sources

### Primary Source Table
```sql
moz-fx-data-shared-prod.telemetry_stable.main_v5
```

### Aggregation Logic

**Client Filtering:**
- Excludes overactive clients (>150,000 pings/day)
- Excludes clients with >3,000,000 addon entries
- Filters to `normalized_app_name = 'Firefox'` only

**Aggregation Strategies:**
- **Counters:** `SUM` (e.g., `crashes_*_sum`, `*_count_sum`)
- **Configuration:** `mode_last` (most recent non-null value)
- **Continuous Metrics:** `AVG` for means, `MAX` for maximums
- **Arrays:** Custom aggregation UDFs (e.g., `aggregate_active_addons`, `aggregate_search_counts`)

**Special Fields:**
- `active_hours_sum`: Derived from `active_ticks / (3600/5)` summed across pings
- `profile_age_in_days`: Calculated from `profile_creation_date` to `subsession_start_date`
- `search_counts`: Complex aggregation splitting engine and source from keyed histogram keys

## Update Schedule

**DAG:** `bqetl_main_summary`
**Frequency:** Daily
**Start Date:** 2019-11-05
**Typical Run Time:** Completes by ~04:00 UTC

**Downstream Dependencies:**
- `jetstream` DAG (wait_for_clients_daily, +2h delay)
- `operational_monitoring` DAG (wait_for_clients_daily, +2h delay)
- `parquet_export` DAG (wait_for_clients_daily, +1h delay)

**Partition Management:**
- Partitioned by `submission_date` (daily)
- Partition filter **required** for all queries
- 775-day retention (automatic deletion of older partitions)

## Usage Examples

### Example 1: Daily Active Users (DAU)
```sql
SELECT
  submission_date,
  normalized_channel,
  COUNT(DISTINCT client_id) AS dau
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date = '2024-01-15'
  AND normalized_channel = 'release'
GROUP BY
  submission_date,
  normalized_channel
```

### Example 2: Search Revenue Metrics
```sql
SELECT
  submission_date,
  country,
  SUM(search_with_ads_count_all) AS searches_with_ads,
  SUM(ad_clicks_count_all) AS ad_clicks,
  SAFE_DIVIDE(
    SUM(ad_clicks_count_all),
    SUM(search_with_ads_count_all)
  ) AS ad_click_through_rate
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
  AND country IN ('US', 'GB', 'CA', 'DE', 'FR')
GROUP BY
  submission_date,
  country
```

### Example 3: Feature Adoption Over Time
```sql
SELECT
  submission_date,
  COUNTIF(e10s_enabled) AS clients_with_e10s,
  COUNTIF(fxa_configured) AS clients_with_fxa,
  COUNTIF(sync_configured) AS clients_with_sync,
  COUNT(DISTINCT client_id) AS total_clients,
  COUNTIF(e10s_enabled) / COUNT(DISTINCT client_id) AS e10s_adoption_rate
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

### Example 4: Experiment Analysis
```sql
SELECT
  e.value AS experiment_branch,
  COUNT(DISTINCT c.client_id) AS clients,
  AVG(c.active_hours_sum) AS avg_active_hours,
  AVG(c.scalar_parent_browser_engagement_total_uri_count_sum) AS avg_uri_count
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6` c,
  UNNEST(c.experiments) e
WHERE
  c.submission_date = '2024-01-15'
  AND e.key = 'my-experiment-id'
GROUP BY
  experiment_branch
```

### Example 5: Crash Rate by OS
```sql
SELECT
  os,
  normalized_os_version,
  COUNT(DISTINCT client_id) AS total_clients,
  COUNTIF(crashes_detected_content_sum > 0) AS clients_with_crashes,
  COUNTIF(crashes_detected_content_sum > 0) / COUNT(DISTINCT client_id) AS crash_rate,
  AVG(crashes_detected_content_sum) AS avg_crashes_per_client
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date = '2024-01-15'
  AND normalized_channel = 'release'
  AND os IS NOT NULL
GROUP BY
  os,
  normalized_os_version
HAVING
  total_clients >= 1000  -- Filter for statistical significance
ORDER BY
  crash_rate DESC
```

### Example 6: Hardware Distribution
```sql
SELECT
  CASE
    WHEN memory_mb < 4096 THEN '<4GB'
    WHEN memory_mb < 8192 THEN '4-8GB'
    WHEN memory_mb < 16384 THEN '8-16GB'
    ELSE '16GB+'
  END AS memory_bucket,
  cpu_vendor,
  COUNT(DISTINCT client_id) AS client_count,
  COUNT(DISTINCT client_id) / SUM(COUNT(DISTINCT client_id)) OVER() AS pct_of_total
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date = '2024-01-15'
  AND normalized_channel = 'release'
  AND memory_mb IS NOT NULL
GROUP BY
  memory_bucket,
  cpu_vendor
ORDER BY
  client_count DESC
```

## Related Tables

### Upstream Dependencies
- **`telemetry_stable.main_v5`**: Primary source for all fields
  - Live table with raw main ping data
  - Updated continuously as pings arrive

### Downstream Tables
- **`telemetry.clients_daily`**: User-facing view (recommended for most queries)
  - Adds compatibility layer and documentation
  - May reference `clients_last_seen_joined_v1` (includes event ping data as of Q1 2021)

- **`telemetry_derived.clients_last_seen_v1`**: Rolling 28-day retention view
  - Tracks first/last seen dates and activity patterns
  - More efficient for cohort and retention analysis

- **`telemetry_derived.clients_last_seen_joined_v1`**: Enhanced version with event data
  - Merges clients_daily with event ping metrics
  - Provides more complete user activity picture

### Related Tables
- **`telemetry_derived.main_summary_v4`**: Predecessor table (deprecated)
  - Use clients_daily_v6 for new analyses
  - Historical data may still reference main_summary

- **`telemetry_derived.core_clients_daily_v1`**: Mobile equivalent
  - Similar structure for Firefox mobile products

## Notes

### Performance Best Practices

1. **Always use partition filter:**
   ```sql
   WHERE submission_date = '2024-01-15'  -- Single day
   -- OR
   WHERE submission_date BETWEEN '2024-01-01' AND '2024-01-31'  -- Date range
   ```

2. **Leverage clustering:**
   ```sql
   WHERE normalized_channel = 'release'  -- First clustering column
   AND sample_id < 10  -- Second clustering column (10% sample)
   ```

3. **Use sampling for exploration:**
   ```sql
   WHERE sample_id = 42  -- 1% sample (1 out of 100 buckets)
   ```

4. **Limit date ranges:**
   - Avoid unbounded date ranges
   - Use CTEs or temp tables for complex multi-stage queries
   - Consider `clients_last_seen` for retention analysis (more efficient)

### Data Quality Considerations

1. **Client Activity:**
   - `active_hours_sum` = 0 indicates client submitted pings but had no active usage
   - Check `pings_aggregated_by_this_row` to understand data completeness

2. **Overactive Client Filtering:**
   - Clients with >150k pings/day are excluded (SQL injection, automation, bugs)
   - Represents <0.001% of population but can skew aggregations

3. **NULL Handling:**
   - Many fields nullable; use `COALESCE` or `IFNULL` for default values
   - Configuration fields use `mode_last`, so NULL means never set across all pings

4. **Array Fields:**
   - `search_counts`, `active_addons`, `experiments` require `UNNEST` for analysis
   - Empty arrays `[]` differ from `NULL` (no field vs. field with no entries)

5. **Geographic Data:**
   - Based on IP geolocation; may be inaccurate for VPNs, proxies
   - `country` is most reliable; city/subdivision have higher error rates

6. **Time Zones:**
   - `submission_date` is server-side (UTC-based)
   - `timezone_offset` captures client's timezone (minutes from UTC)

### Migration Notes

- **From main_summary_v4:**
  - Field names mostly unchanged
  - New fields added for recent Firefox features
  - Improved aggregation logic (fewer edge case bugs)
  - Query patterns remain similar

- **To clients_last_seen_joined_v1:**
  - Includes event ping metrics (interactions, Glean data)
  - Recommended for comprehensive user activity analysis (Q1 2021+)

### Getting Help

- **Documentation:** https://docs.telemetry.mozilla.org/
- **Looker:** Pre-built dashboards at https://mozilla.cloud.looker.com/
- **Community:** #data-help on Mozilla Slack
- **SQL Observatory:** https://sql.telemetry.mozilla.org/

### Version History

- **v6 (current):** Enhanced schema, improved aggregations, added recent features
- **v5:** Source migrated from main_v4 to main_v5
- **v4 and earlier:** See bigquery-etl commit history

## Owners

- Primary: ascholtz@mozilla.com
- Team: Data Engineering & Data Science
- Slack: #data-help

## Additional Metadata

- **Application:** Firefox Desktop
- **Schedule:** Daily via Airflow DAG `bqetl_main_summary`
- **Table Type:** Client-level aggregation
- **Labels:** `application:firefox`, `schedule:daily`, `table_type:client_level`
