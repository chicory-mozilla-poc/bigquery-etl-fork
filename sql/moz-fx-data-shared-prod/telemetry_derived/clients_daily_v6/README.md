# Clients Daily v6

## Overview

The `clients_daily_v6` table is a foundational dataset that provides a daily aggregated view of Firefox Desktop client activity and characteristics. This table consolidates multiple telemetry pings received from each unique Firefox client on a given day into a single row, making it the primary source for client-level daily analysis.

## Purpose

This table serves as the core dataset for understanding Firefox Desktop client behavior, system configuration, feature usage, and engagement metrics on a per-client, per-day basis. It aggregates data from the `telemetry_stable.main_v5` ping table, applying intelligent aggregation strategies (sums, averages, mode-last) to distill hundreds of daily pings per client into meaningful daily summaries.

## Data Sources

### Primary Source
- **Table**: `moz-fx-data-shared-prod.telemetry_stable.main_v5`
- **Ping Type**: `main` pings from Firefox Desktop
- **Filter**: `normalized_app_name = 'Firefox'`

### Data Flow
1. Raw `main` pings arrive in `telemetry_stable.main_v5`
2. Pings are filtered for Firefox Desktop clients with valid document IDs
3. Overactive clients (>150,000 pings/day or >3M active addons) are excluded to prevent aggregation errors
4. Data is aggregated by `client_id` and `submission_date`
5. Various aggregation strategies are applied based on metric type:
   - **SUM**: Counters, event counts, usage metrics
   - **AVG/MEAN**: Duration metrics, histogram averages
   - **MODE_LAST**: Configuration settings (most recent value wins)
   - **LOGICAL_OR**: Boolean flags (true if ever true that day)

## Update Schedule

- **Frequency**: Daily
- **DAG**: `bqetl_main_summary`
- **Partition**: By `submission_date` (required partition filter)
- **Retention**: 775 days (~2 years)
- **Start Date**: 2019-11-05

### Dependencies
This table is a critical dependency for downstream processes:
- **Jetstream** experiments analysis (2h execution delta)
- **Operational Monitoring** dashboards (2h execution delta)
- **Parquet Export** for external data sharing (1h execution delta)
- **clients_last_seen_joined_v1** (merged with event ping data)

## Schema Overview

The table contains 360+ columns organized into these categories:

### 1. **Identifiers & Dates** (4 columns)
- `submission_date`: Server-side date when pings were received
- `client_id`: Unique client identifier (UUID)
- `sample_id`: Sampling bucket (0-99) for data sampling
- `profile_group_id`: Profile group identifier

### 2. **System Hardware** (~30 columns)
- CPU details (cores, vendor, model, speed, cache)
- Memory configuration
- Graphics features and status
- OS information and version
- Apple hardware model ID (Mac only)

### 3. **Application Info** (~15 columns)
- Firefox version, build ID, channel
- Distribution and partner information
- Update settings and previous build

### 4. **Geography & Network** (~10 columns)
- Country, city, subdivisions (from IP geolocation)
- ISP name and organization
- Geo database version

### 5. **Usage & Engagement** (~40 columns)
- Active hours and session metrics
- URI counts and unique domains visited
- Tab and window management
- Search counts by source
- Bookmark and history metrics

### 6. **Browser Configuration** (~50 columns)
- Default browser status
- Search engine settings (normal and private mode)
- Privacy and security settings
- Locale and internationalization
- Addon/extension settings
- User preferences

### 7. **Addons & Extensions** (5 columns)
- Active addons list with detailed metadata
- Addon counts and compatibility info
- System/web extension flags

### 8. **Search Metrics** (~80 columns)
- Search counts by source (urlbar, searchbar, about:home, etc.)
- Search with ads impressions and clicks
- Quick Suggest interactions
- Contextual services metrics
- Ad clicks by source

### 9. **Developer Tools Usage** (~40 columns)
- Individual devtools panel open counts
- Accessibility tools usage
- Performance profiler usage
- Network monitor, debugger, console activity

### 10. **Performance & Stability** (~30 columns)
- Crash counts by process type (main, content, plugin)
- Crash submit attempts and successes
- Plugin hangs and abort counts
- SSL handshake results
- Client clock skew and submission latency

### 11. **Features & Interactions** (~50 columns)
- URL bar interactions (autofill, bookmarks, history picks)
- Search mode usage
- Content interaction patterns
- Text recognition usage
- Media playback time
- Browser migration metrics (from Chrome, Edge, Safari)
- Taskbar and desktop integration (Windows)
- Private browsing usage

### 12. **Experiments & Telemetry** (~10 columns)
- Active experiment assignments
- Telemetry event counts
- Search cohort assignment
- FxA and Sync configuration

## Key Features

### Intelligent Aggregation
- **Temporal Ordering**: Most configuration fields use `mode_last` to capture the most recent state
- **Overactive Client Filtering**: Excludes clients with abnormal ping volumes that could skew analysis
- **NULL Handling**: Careful handling of NULL values to prevent loss of meaningful data

### Data Quality
- **Partition Requirement**: Queries must include a partition filter on `submission_date` for performance
- **Clustering**: Optimized by `normalized_channel` and `sample_id` for common query patterns
- **Document Deduplication**: Uses `first_document_id` to enable deduplication if needed

## Common Usage Examples

### Basic Daily Active Users (DAU)
```sql
SELECT
  submission_date,
  COUNT(DISTINCT client_id) AS dau
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date = '2024-01-01'
GROUP BY
  submission_date
```

### Channel Distribution
```sql
SELECT
  submission_date,
  normalized_channel,
  COUNT(DISTINCT client_id) AS clients
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY
  submission_date,
  normalized_channel
```

### Active Hours by Country
```sql
SELECT
  country,
  SUM(active_hours_sum) AS total_active_hours,
  AVG(active_hours_sum) AS avg_active_hours_per_client
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date = '2024-01-01'
  AND active_hours_sum IS NOT NULL
GROUP BY
  country
ORDER BY
  total_active_hours DESC
LIMIT 20
```

### Search Behavior Analysis
```sql
SELECT
  submission_date,
  SUM(search_count_all) AS total_searches,
  SUM(search_count_urlbar) AS urlbar_searches,
  SUM(search_count_searchbar) AS searchbar_searches,
  SUM(search_with_ads_count_all) AS searches_with_ads,
  SUM(ad_clicks_count_all) AS ad_clicks
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY
  submission_date
ORDER BY
  submission_date
```

### System Configuration Distribution
```sql
SELECT
  os,
  memory_mb,
  cpu_cores,
  COUNT(DISTINCT client_id) AS client_count
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date = '2024-01-01'
GROUP BY
  os,
  memory_mb,
  cpu_cores
ORDER BY
  client_count DESC
LIMIT 50
```

## Important Notes

### Best Practices
1. **Always use partition filters** on `submission_date` for query performance
2. **Use sampling** via `sample_id` for exploratory analysis (e.g., `WHERE sample_id = 0` gives 1% sample)
3. **Consider user privacy** - client_id is present but should be used responsibly
4. **Account for NULL values** - many fields can be NULL if not reported in telemetry

### Limitations
- Excludes overactive clients (>150K pings/day) to prevent data quality issues
- Some fields deprecated and set to NULL (e.g., `active_experiment_id`)
- Requires understanding of aggregation strategies to interpret metrics correctly
- Historical data before 2019-11-22 uses a different query version

### Data Freshness
- Typically available within 2-3 hours after UTC day boundary
- Downstream consumers wait 1-2 hours after this table updates

## Related Tables

- **telemetry.clients_daily**: User-facing view that references this table
- **telemetry_derived.clients_last_seen_v1**: Longitudinal view built from this table
- **telemetry_derived.clients_last_seen_joined_v1**: Merged with event ping data
- **telemetry_stable.main_v5**: Raw source data
- **telemetry.main**: User-facing view of main pings

## Contact & Support

- **Owner**: ascholtz@mozilla.com
- **GitHub Issues**: [bigquery-etl repository](https://github.com/mozilla/bigquery-etl/issues)
- **Slack**: #data-help channel in Mozilla Slack

## Change History

- **2019-11-05**: Initial v6 schema deployment
- **2019-11-22**: Added new fields to main_v4 schema (active_addons fields, measurements)
- **2021 Q1**: User-facing view transitioned to clients_last_seen_joined_v1
- **Ongoing**: Regular additions of new metrics and telemetry probes
