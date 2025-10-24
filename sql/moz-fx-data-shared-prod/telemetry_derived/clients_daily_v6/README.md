# clients_daily_v6

## Overview

The `clients_daily_v6` table provides a comprehensive daily aggregation of Firefox Desktop telemetry data at the client level. This table is one of the most widely-used datasets in Mozilla's data ecosystem, serving as the foundation for user behavior analysis, product health monitoring, and business intelligence reporting.

**Table:** `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`

**Source:** `moz-fx-data-shared-prod.telemetry_stable.main_v5`

**Grain:** One row per client per day (based on `submission_date`)

## Purpose

This table aggregates multiple telemetry pings from individual Firefox clients into a single daily summary record. It consolidates data across:
- Browser engagement metrics (active hours, URI counts, tab/window usage)
- System and hardware configuration
- Search activity and search engine usage
- Crash and stability metrics
- Add-on and experiment participation
- Developer tools usage
- Geographic and demographic information

## Key Characteristics

- **Daily Aggregation:** Multiple pings per client per day are rolled up using various aggregation strategies (sum, average, mode_last)
- **Overactive Client Filtering:** Excludes clients with >150,000 pings/day or >3,000,000 active addons to prevent aggregation errors
- **Historical Coverage:** Available from 2019-11-22 onwards (when key schema fields were added to main_v5)
- **Typical Volume:** Millions of rows per day (one per active Firefox Desktop client)

## Schema Summary

The table contains **409 columns** organized into the following categories:

### Core Identifiers
- `submission_date` - Date when telemetry was received (partition key)
- `client_id` - Unique client identifier (UUID)
- `sample_id` - Sample bucket (0-99) for consistent sampling
- `document_id` - First document ID seen for the day
- `profile_group_id` - Profile group identifier

### Engagement & Usage Metrics
- `active_hours_sum` - Total active usage hours (based on 5-second ticks)
- `subsession_hours_sum` - Total session duration in hours
- `scalar_parent_browser_engagement_total_uri_count_sum` - Total URIs visited
- `scalar_parent_browser_engagement_unique_domains_count_max` - Peak unique domains visited
- `sessions_started_on_this_day` - Number of browsing sessions initiated

### Search Metrics
- `search_counts` - Array of search counts by engine and source
- `search_count_all` - Total search count across all sources
- `search_with_ads_count_all` - Searches that displayed ads
- `ad_clicks_count_all` - Clicks on search ads

### System & Environment
- `os`, `os_version`, `normalized_os_version` - Operating system information
- `cpu_*` fields - CPU architecture, cores, speed, vendor
- `memory_mb` - System RAM in megabytes
- `gfx_features_*` - Graphics feature support status

### Browser Configuration
- `app_version`, `app_build_id` - Firefox version information
- `channel`, `normalized_channel` - Release channel (release, beta, nightly, etc.)
- `locale` - Browser language/locale setting
- `default_search_engine` - Default search engine name

### Geography & Network
- `country`, `city`, `geo_subdivision1`, `geo_subdivision2` - Geographic location
- `isp_name`, `isp_organization` - Internet service provider

### Stability & Performance
- `crashes_detected_*_sum` - Crash counts by process type
- `crash_submit_*_sum` - Crash report submission metrics
- `aborts_*_sum` - Process abort counts

### Features & Experiments
- `experiments` - Active experiments and branches
- `active_addons` - Installed and enabled add-ons
- `fxa_configured`, `sync_configured` - Firefox Account and Sync status

### User Preferences
- `user_pref_*` fields - Selected user preference settings
- `is_default_browser` - Whether Firefox is the default browser
- `telemetry_enabled` - Telemetry opt-in status

## Data Sources

**Primary Source:**
- `moz-fx-data-shared-prod.telemetry_stable.main_v5` - Raw Firefox telemetry pings

**Aggregation Logic:**
- Filters for `normalized_app_name = 'Firefox'` (Desktop only)
- Excludes null document_ids
- Excludes overactive clients (>150k pings/day)
- Groups by `client_id` and `DATE(submission_timestamp)`

**Key Transformations:**
- Histogram extraction using UDFs (e.g., `mozfun.hist.extract`, `mozfun.hist.mean`)
- Search count parsing from keyed histograms
- Active addon aggregation and deduplication
- Mode-last selection for dimensional attributes (takes most recent value)
- Summation for counter metrics
- Averaging for gauge metrics

## Update Schedule

- **Frequency:** Daily
- **Scheduling:** Automated via Airflow DAG
- **Partition Field:** `submission_date` (DATE)
- **Backfill Availability:** Historical data available from 2019-11-22

**Data Freshness:**
- Data for date `D` is typically available by 06:00 UTC on day `D+1`
- Uses server-side `submission_timestamp` (when Mozilla receives the ping), not client-side timestamps

## Usage Examples

### Example 1: Daily Active Users (DAU)
```sql
SELECT
  submission_date,
  COUNT(DISTINCT client_id) AS dau
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY
  submission_date
ORDER BY
  submission_date
```

### Example 2: Average Active Hours by Channel
```sql
SELECT
  submission_date,
  normalized_channel,
  AVG(active_hours_sum) AS avg_active_hours
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date = '2024-01-15'
  AND active_hours_sum > 0
GROUP BY
  submission_date,
  normalized_channel
```

### Example 3: Search Activity by Engine
```sql
SELECT
  submission_date,
  sc.engine,
  SUM(sc.count) AS total_searches
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`,
  UNNEST(search_counts) AS sc
WHERE
  submission_date = '2024-01-15'
GROUP BY
  submission_date,
  sc.engine
ORDER BY
  total_searches DESC
```

### Example 4: New Profile Cohort Analysis
```sql
SELECT
  DATE(SAFE.TIMESTAMP(profile_creation_date)) AS cohort_date,
  COUNT(DISTINCT client_id) AS new_profiles
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date = DATE(SAFE.TIMESTAMP(profile_creation_date))
  AND submission_date BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY
  cohort_date
ORDER BY
  cohort_date
```

### Example 5: Sampling for Large Queries
```sql
-- Use sample_id for consistent 1% sample
SELECT
  submission_date,
  COUNT(DISTINCT client_id) AS sampled_dau
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date = '2024-01-15'
  AND sample_id = 0  -- 1% sample
```

## Important Considerations

### Sampling
- Use `sample_id` (0-99) for consistent client sampling across queries
- Each sample_id represents approximately 1% of the population
- Sample assignment is stable per client_id

### Aggregation Strategy
- **Counters (SUM):** Metrics like crashes, searches, URI counts
- **Mode Last:** Most recent value for configuration fields (OS, version, locale)
- **Average:** Calculated metrics like bookmarks, pages, active addons count
- **Max:** Peak values like concurrent tabs/windows

### Data Quality Notes
- `active_hours_sum` is derived from active_ticks (5-second intervals), not wall-clock time
- `subsession_hours_sum` includes idle time and may exceed 24 hours if multiple devices sync
- Search counts exclude searches that don't send SAP attribution
- Geographic fields based on GeoIP lookup, may have inaccuracies

### Performance Tips
- Always filter by `submission_date` (partition key) for optimal performance
- Use `sample_id` for exploratory analysis to reduce data scanned
- Pre-aggregate before joining to other tables when possible
- Avoid `SELECT *` due to 409 columns; specify needed columns

## Related Tables

- **clients_last_seen_v1** - Longitudinal view showing client activity over 28-day windows
- **main_v5** - Raw source pings with full granularity
- **clients_first_seen_v1** - First appearance date for each client
- **main_summary_v4** - Deprecated predecessor (use clients_daily_v6 instead)
- **events_daily_v1** - Event telemetry aggregated daily
- **search_clients_engines_sources_daily_v1** - Detailed search metrics

## Notes

- **Historical Context:** This is version 6 of the clients_daily schema. Previous versions had different field sets and aggregation logic.
- **Desktop Only:** This table only contains Firefox Desktop data. Mobile products (Fenix, iOS) have separate tables.
- **Attribution Data:** The `attribution` struct contains marketing attribution for installations (source, medium, campaign, etc.)
- **Experiment Tracking:** The `experiments` field shows active experiments. For detailed experiment analysis, use dedicated experiment tables.
- **Privacy:** All data is aggregated and anonymized. The `client_id` is a randomly-generated UUID, not personally identifiable.

## Schema Change History

- **v6 (2019-11-22):** Current version, reads from main_v5, includes addon details and active_ticks from simple_measurements
- **v5 and earlier:** Deprecated versions with different schemas

## Support & Documentation

- **DTMO:** https://docs.telemetry.mozilla.org/datasets/batch_view/clients_daily/reference.html
- **Looker:** Pre-built dashboards available in Mozilla's Looker instance
- **Data Catalog:** Search for "clients_daily" in Mozilla's data catalog
- **Questions:** Ask in #data-help on Mozilla Slack

---

**Last Updated:** 2025-01-19
**Maintained By:** Data Engineering Team
**ETL Source:** https://github.com/mozilla/bigquery-etl