# Clients Daily v6

## Overview

The `clients_daily_v6` table provides a comprehensive daily aggregation of Firefox Desktop telemetry data, consolidating all `main` pings from each client into a single row per client per day. This table serves as a foundational dataset for analyzing Firefox Desktop user behavior, browser performance, engagement metrics, and system characteristics.

## Table Details

- **Full Table Name**: `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
- **User-Facing View**: `telemetry.clients_daily` (recommended for most queries)
- **Partition Field**: `submission_date` (DATE, daily partitioning with 775-day retention)
- **Clustering**: `normalized_channel`, `sample_id`
- **Granularity**: One row per `client_id` per `submission_date`

## Data Sources and Dependencies

### Primary Source
- **`telemetry_stable.main_v5`**: The main Firefox Desktop telemetry ping containing environment information, performance metrics, usage statistics, and system data

### Processing Logic
The table aggregates multiple telemetry pings from each client daily using various aggregation strategies:
- **Sums**: Counts, durations, and event totals across all pings
- **Means/Averages**: Performance metrics like active hours, bookmarks count
- **Mode Last**: Most recent common value for browser settings and configurations
- **Max**: Maximum values for concurrent tabs, windows, and domain counts
- **Arrays**: Search counts, experiments, and addon details

### Data Quality Filters
- Excludes overactive clients (>150,000 pings/day or >3,000,000 active addons aggregate)
- Filters for Firefox Desktop only (`normalized_app_name = 'Firefox'`)
- Requires non-null `document_id`

## Update Schedule

- **Frequency**: Daily
- **DAG**: `bqetl_main_summary`
- **Schedule**: Runs daily, typically completing by early morning UTC
- **Downstream Dependencies**:
  - Jetstream (2-hour execution delta)
  - Operational Monitoring (2-hour execution delta)
  - Parquet Export (1-hour execution delta)

## Schema Overview

The table contains 350+ columns organized into the following categories:

### Identifiers & Temporal Fields
- `submission_date`: Server-side date when telemetry was received (partition key)
- `client_id`: Unique UUID identifying the Firefox installation
- `sample_id`: Sample identifier (0-99) for stratified sampling
- `document_id`: First document ID from pings aggregated in the row
- `profile_group_id`: UUID identifying the profile group

### Client Environment & System Information
- Operating system details (OS name, version, Windows build numbers)
- Hardware specifications (CPU, memory, GPU features)
- Geographic data (country, city, ISP)
- Browser version and build information
- Partner and distribution details

### Usage & Engagement Metrics
- `active_hours_sum`: Total active browsing time
- `subsession_hours_sum`: Total session time including inactive periods
- `scalar_parent_browser_engagement_total_uri_count_sum`: Total URIs loaded
- `scalar_parent_browser_engagement_unique_domains_count_max`: Maximum unique domains visited
- `scalar_parent_browser_engagement_tab_open_event_count_sum`: Total tabs opened
- `sessions_started_on_this_day`: Number of browsing sessions initiated

### Search Metrics
- `search_counts`: Array of search counts by engine and source
- `search_count_*`: Aggregated counts by source (urlbar, searchbar, about:home, etc.)
- `ad_clicks`: Search ad click counts by engine
- `search_with_ads`: Search result page impressions with ads
- `default_search_engine`: Primary configured search engine

### Browser Features & Settings
- `is_default_browser`: Whether Firefox is set as the system default
- `e10s_enabled`: Multi-process architecture enabled status
- `fxa_configured`: Firefox Accounts configured
- `sync_configured`: Sync service enabled
- `locale`: User interface language preference
- `attribution`: Installation attribution data (source, campaign, etc.)

### Add-ons & Extensions
- `active_addons`: Detailed array of installed and enabled add-ons
- `active_addons_count_mean`: Average number of active add-ons

### Performance & Stability
- `crashes_detected_content_sum`: Content process crashes detected
- `crash_submit_success_main_sum`: Successfully submitted main process crash reports
- `plugin_hangs_sum`: Plugin hang incidents
- `first_paint_mean`: Average time to first paint
- `places_bookmarks_count_mean`: Average bookmark count
- `places_pages_count_mean`: Average browsing history size

### Developer Tools Usage
- Multiple `histogram_parent_devtools_*_opened_count_sum` columns tracking devtools panel usage
- `devtools_toolbox_opened_count_sum`: Total devtools openings

### Experiments & Studies
- `experiments`: Array of active experiments and their branches
- `active_experiment_id` / `active_experiment_branch`: Primary active experiment

### Contextual Services & Quick Suggest
- `contextual_services_quicksuggest_*`: Metrics for Firefox Suggest feature
- `contextual_services_topsites_*`: Top sites engagement metrics

### Additional Features
- Migration data from other browsers (Chrome, Edge, Safari)
- Text recognition feature usage
- Private browsing indicators
- Sidebar and library usage metrics
- Accessibility theme settings
- Media playback time (audio/video)

## Usage Examples

### Example 1: Daily Active Users by Channel
```sql
SELECT
  submission_date,
  normalized_channel,
  COUNT(DISTINCT client_id) AS dau
FROM
  `moz-fx-data-shared-prod.telemetry.clients_daily`
WHERE
  submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
GROUP BY
  submission_date,
  normalized_channel
ORDER BY
  submission_date DESC,
  normalized_channel
```

### Example 2: Average Active Hours by Country
```sql
SELECT
  country,
  AVG(active_hours_sum) AS avg_active_hours,
  COUNT(DISTINCT client_id) AS client_count
FROM
  `moz-fx-data-shared-prod.telemetry.clients_daily`
WHERE
  submission_date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
  AND country != '??'
GROUP BY
  country
HAVING
  client_count >= 100
ORDER BY
  client_count DESC
LIMIT 20
```

### Example 3: Search Engine Distribution
```sql
SELECT
  default_search_engine,
  COUNT(DISTINCT client_id) AS clients,
  SUM(search_count_all) AS total_searches
FROM
  `moz-fx-data-shared-prod.telemetry.clients_daily`
WHERE
  submission_date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
  AND search_count_all > 0
GROUP BY
  default_search_engine
ORDER BY
  clients DESC
```

### Example 4: Browser Version Adoption
```sql
SELECT
  app_version,
  COUNT(DISTINCT client_id) AS clients,
  ROUND(COUNT(DISTINCT client_id) / SUM(COUNT(DISTINCT client_id)) OVER () * 100, 2) AS pct
FROM
  `moz-fx-data-shared-prod.telemetry.clients_daily`
WHERE
  submission_date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
  AND normalized_channel = 'release'
GROUP BY
  app_version
ORDER BY
  clients DESC
LIMIT 10
```

### Example 5: Add-on Usage Analysis
```sql
SELECT
  addon.addon_id,
  addon.name,
  COUNT(DISTINCT client_id) AS install_count
FROM
  `moz-fx-data-shared-prod.telemetry.clients_daily`,
  UNNEST(active_addons) AS addon
WHERE
  submission_date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
  AND addon.addon_id IS NOT NULL
GROUP BY
  addon.addon_id,
  addon.name
ORDER BY
  install_count DESC
LIMIT 20
```

## Related Tables

- **`telemetry_derived.clients_last_seen_v1`**: Contains historical client activity across multiple days, includes 1-day, 7-day, and 28-day activity windows
- **`telemetry_derived.clients_last_seen_joined_v1`**: Merges `clients_last_seen` with event ping data
- **`telemetry.main`**: Raw, non-aggregated main ping data (use for investigating individual pings)
- **`telemetry_stable.main_v5`**: Stable view of main pings (source for clients_daily)
- **`telemetry_derived.main_summary_v4`**: Legacy aggregated table (deprecated, use clients_daily instead)

## Important Notes

### Partition Filters Required
Always include a `submission_date` filter in your queries to avoid scanning the entire table:
```sql
WHERE submission_date >= '2024-01-01'
```

### Using Sample ID for Development
For faster query development, filter to a single sample_id (1% sample):
```sql
WHERE sample_id = 42
```

### Data Freshness
- Data for date `D` is typically available by `D+1` at approximately 02:00 UTC
- Allow extra time for downstream dependencies

### Historical Backfill
For dates on or before 2019-11-22, the table was backfilled from `main_summary_v4` using a different query version. Some fields may have limited or missing data for these earlier dates.

### Null Handling
Many columns may be `NULL` when:
- The feature is not applicable to the client's Firefox version
- The client has disabled telemetry for specific features
- The data was not available in the source pings

### Client ID Changes
A single user may have multiple `client_id` values over time due to:
- Firefox reinstallation
- Profile resets or profile creation
- Telemetry opt-out followed by opt-in

Use `clients_last_seen` for retention and cohort analysis requiring longitudinal tracking.

## Owners & Contact

- **Primary Owner**: ascholtz@mozilla.com
- **Application**: Firefox Desktop
- **Table Type**: Client-level aggregation

## Additional Resources

- [BigQuery ETL Repository](https://github.com/mozilla/bigquery-etl)
- [Firefox Data Documentation](https://docs.telemetry.mozilla.org/)
- [Metric Hub](https://mozilla.acryl.io/)
- [Data Catalog (Seshat)](https://mozilla.acryl.io/)

---

*Last Updated: 2024*
*Table Version: v6*
