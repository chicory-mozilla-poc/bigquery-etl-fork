# clients_daily_v6

## Overview

The `clients_daily_v6` table is a foundational dataset for Firefox desktop telemetry analysis, providing comprehensive daily aggregations of client activity and browser metrics. This table consolidates multiple telemetry pings per client per day into a single row, making it ideal for understanding user behavior, system configurations, and feature adoption at scale.

**Table**: `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`

## Key Characteristics

| Attribute | Description |
|-----------|-------------|
| **Grain** | One row per `client_id` per `submission_date` |
| **Source Table** | `moz-fx-data-shared-prod.telemetry_stable.main_v5` |
| **Update Frequency** | Daily (partitioned by submission_date) |
| **Application Scope** | Firefox desktop only (`normalized_app_name = 'Firefox'`) |
| **Data Quality** | Excludes overactive clients (>150,000 pings/day or >3M active addons) |
| **Row Count (typical)** | Millions of rows per day (varies by active user base) |

## Data Sources and ETL Logic

### Source Data
- **Primary Source**: `telemetry_stable.main_v5` - Firefox main telemetry pings
- **Date Filter**: `DATE(submission_timestamp) = @submission_date`
- **Application Filter**: `normalized_app_name = 'Firefox'` AND `document_id IS NOT NULL`

### Aggregation Strategy

The ETL process follows a multi-stage approach:

1. **Base Extraction** (`base` CTE): Extracts raw telemetry data with initial transformations
   - Histogram extractions using mozfun UDFs
   - Keyed histogram summarizations
   - Array batching to optimize UDF invocations

2. **Overactive Client Filtering** (`overactive` CTE): Identifies and excludes problematic clients
   - Clients with >150,000 pings in a single day
   - Clients with >3,000,000 total active addons across all pings
   - Purpose: Prevent aggregation overflow errors

3. **Client Summary** (`clients_summary` CTE): Per-ping transformations
   - Flattens nested structures from telemetry
   - Applies type casting and data cleaning
   - Extracts scalar values from histograms

4. **Daily Aggregation** (`aggregates` CTE): Groups by client_id and submission_date
   - **Sum aggregations**: Counts, crashes, search metrics, usage hours
   - **Average aggregations**: Active hours, clock skew, bookmark/page counts
   - **Mode_last**: Most recent non-null value for configuration fields
   - **Max/Min**: Subsession counters, engagement peaks
   - **Custom UDFs**: Active addons aggregation, search counts mapping

5. **Final Selection**: Unpacks complex aggregations into final schema

### Aggregation Functions Used

- `SUM()`: Cumulative metrics (crashes, searches, active hours)
- `AVG()`: Mean values (latency, bookmarks count)
- `MAX()/MIN()`: Peak values and ranges
- `mozfun.stats.mode_last()`: Most recent non-null configuration
- `mozfun.map.sum()`: Keyed scalar aggregations
- `ARRAY_AGG()`: Collecting values for UDF processing
- `COUNTIF()`: Boolean flag counting (launch methods)
- `LOGICAL_OR()`: Any-true across pings

## Schema Organization

The table contains **363 columns** organized into the following categories:

### Core Identifiers (4 columns)
- `submission_date`: Server-side receipt date (partitioning key)
- `client_id`: Unique client identifier (UUID)
- `sample_id`: 0-99 sampling bucket
- `profile_group_id`: UUID for profile group identification

### System & Hardware (35+ columns)
- Operating system details (`os`, `os_version`, `normalized_os_version`)
- CPU specifications (`cpu_*` fields)
- Memory configuration (`memory_mb`)
- GPU/graphics features (`gfx_features_*`)
- Windows-specific (`windows_build_number`, `windows_ubr`, `is_wow64`)
- Apple-specific (`apple_model_id`)

### Browser Configuration (30+ columns)
- Application version info (`app_version`, `app_build_id`)
- Update settings (`update_enabled`, `update_auto_download`)
- Environment build details (`env_build_*`)
- E10s and sandbox configuration
- Attribution data (campaign, source, medium)

### Engagement & Usage (25+ columns)
- Active hours (`active_hours_sum`)
- Total/subsession hours (`subsession_hours_sum`)
- URI counts (`scalar_parent_browser_engagement_total_uri_count_sum`)
- Tab and window metrics
- Session started count
- Profile age and creation date

### Search Behavior (50+ columns)
- Search counts by access point (urlbar, about:home, context menu, etc.)
- Search with ads tracking
- Ad click metrics
- Default search engine configuration
- Quick suggest interactions (sponsored/non-sponsored)
- Search mode tracking

### Crashes & Stability (20+ columns)
- Main process crashes
- Content process crashes
- Plugin crashes and hangs
- Crash submission attempts and successes
- SSL handshake results
- Shutdown kills

### Add-ons & Extensions (10+ columns)
- Active addons array with details
- Addon count metrics
- Blocklist settings
- Extension-related scalars

### Developer Tools (50+ columns)
- DevTools panel open counts (Inspector, Console, Debugger, etc.)
- Accessibility features usage
- CSS selector copying
- Various devtools interaction metrics

### Geolocation & ISP (6 columns)
- Country, city, subdivisions
- ISP name and organization
- Geo database version

### Localization & Internationalization (10+ columns)
- Locale settings
- Accept languages
- Available/requested locales
- Regional preferences

### Experiments & Features (5+ columns)
- Active experiments array
- Search cohort
- FxA and Sync configuration
- Feature flag tracking

### User Preferences (15+ columns)
- Search-related prefs (`user_pref_browser_search_*`)
- Urlbar behavior prefs
- Quick suggest preferences
- Newtab settings

### Migration Metrics (12 columns)
- Browser migration quantities (Chrome, Edge, Safari)
- Bookmarks, history, and logins migration counts

### Media & Content (2 columns)
- Audio playback time
- Video playback time

### Places/History (8 columns)
- Bookmarks count
- Pages count
- Previous day visits
- Library and searchbar search metrics

### Other Metrics (40+ columns)
- Telemetry event counts
- Contextual services metrics
- UI interaction tracking
- Text recognition features
- Sidebar and library usage
- Profile selection reasons
- Launch method tracking

## Common Query Patterns

### Daily Active Users (DAU)
```sql
SELECT
  submission_date,
  COUNT(DISTINCT client_id) AS dau
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date = '2024-01-15'
GROUP BY
  submission_date
```

### User Engagement Analysis
```sql
SELECT
  submission_date,
  APPROX_QUANTILES(active_hours_sum, 100)[OFFSET(50)] AS median_active_hours,
  APPROX_QUANTILES(scalar_parent_browser_engagement_total_uri_count_sum, 100)[OFFSET(50)] AS median_uri_count
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY
  submission_date
ORDER BY
  submission_date
```

### Operating System Distribution
```sql
SELECT
  os,
  normalized_os_version,
  COUNT(DISTINCT client_id) AS client_count
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date = '2024-01-15'
GROUP BY
  os, normalized_os_version
ORDER BY
  client_count DESC
```

### Search Behavior by Source
```sql
SELECT
  submission_date,
  SUM(search_count_urlbar) AS urlbar_searches,
  SUM(search_count_searchbar) AS searchbar_searches,
  SUM(search_count_newtab) AS newtab_searches,
  SUM(search_count_abouthome) AS abouthome_searches
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-07'
GROUP BY
  submission_date
ORDER BY
  submission_date
```

### Crash Rate Analysis
```sql
SELECT
  submission_date,
  SUM(crashes_detected_content_sum) AS content_crashes,
  SUM(crashes_detected_plugin_sum) AS plugin_crashes,
  COUNT(DISTINCT client_id) AS total_clients,
  SUM(crashes_detected_content_sum) / COUNT(DISTINCT client_id) AS crashes_per_client
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY
  submission_date
ORDER BY
  submission_date
```

### Add-on Installation Analysis
```sql
SELECT
  submission_date,
  AVG(active_addons_count_mean) AS avg_addons_per_client,
  APPROX_QUANTILES(active_addons_count_mean, 100)[OFFSET(50)] AS median_addons
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY
  submission_date
ORDER BY
  submission_date
```

### Experiment Participation
```sql
SELECT
  submission_date,
  exp.key AS experiment_id,
  exp.value AS experiment_branch,
  COUNT(DISTINCT client_id) AS client_count
FROM
  `moz-fx-data-shared-prod.telemetry_derived.clients_daily_v6`,
  UNNEST(experiments) AS exp
WHERE
  submission_date = '2024-01-15'
GROUP BY
  submission_date, experiment_id, experiment_branch
ORDER BY
  client_count DESC
```

## Data Quality Considerations

### Strengths
✅ **High Coverage**: Captures majority of Firefox desktop users with telemetry enabled
✅ **Comprehensive**: 363 columns covering system, behavior, and feature usage
✅ **Daily Granularity**: Enables trend analysis and day-over-day comparisons
✅ **Quality Filtering**: Excludes overactive clients that could skew aggregations
✅ **Standardized Aggregations**: Consistent use of mozfun UDFs for reliable metrics

### Limitations
⚠️ **Opt-in Data**: Only includes users with telemetry enabled (excludes opt-out users)
⚠️ **Aggregation Level**: Daily aggregation may mask intra-day patterns
⚠️ **Null Handling**: Many columns can be NULL if not applicable or not collected
⚠️ **Schema Evolution**: Column availability varies by Firefox version and date
⚠️ **Large Table**: High cardinality requires careful query optimization and sampling

### Best Practices
1. **Always filter by `submission_date`**: Partitioning key for query performance
2. **Use `sample_id` for exploratory analysis**: 1% sample = `WHERE sample_id = 0`
3. **Check for NULLs**: Many metrics are NULL when feature not used
4. **Understand aggregation**: Values are already summed/averaged per client per day
5. **Join carefully**: High row count makes joins expensive; prefer smaller dimensions
6. **Consider `normalized_channel`**: Separate release, beta, nightly for analysis

## Related Tables

| Table | Relationship | Use Case |
|-------|--------------|----------|
| `telemetry.main` | Source (ping-level) | Detailed ping-level analysis, sub-day granularity |
| `telemetry.clients_last_seen` | Derived (rolling window) | Retention analysis, multi-day patterns |
| `telemetry.main_summary_v4` | Deprecated predecessor | Historical analysis (pre-2019) |
| `telemetry_derived.clients_first_seen` | Supplemental | New user cohort analysis |
| `search.search_clients_engines_sources_daily` | Complementary | Detailed search metrics |

## Historical Context

### Version History
- **v6 (current)**: Introduced November 2019 with schema enhancements for active_addons fields
- **v4 (deprecated)**: Previous version reading from `main_summary_v4`
- **Backfill Notes**: Data before 2019-11-22 requires different query version (see query comments)

### Schema Changes
- 2019-11-22: Added active_addons foreign_install, user_disabled, version fields
- Ongoing: New scalar and histogram fields added as Firefox telemetry evolves

## Support and Documentation

- **Schema Definition**: See `schema.yaml` in this directory
- **Query Source**: `query.sql` contains full ETL logic
- **Mozilla Documentation**: https://docs.telemetry.mozilla.org/
- **Telemetry Dictionary**: https://dictionary.telemetry.mozilla.org/
- **Data Platform**: https://docs.telemetry.mozilla.org/cookbooks/

## Notes

- This table is automatically updated daily via the BigQuery ETL pipeline
- Data is partitioned by `submission_date` for optimal query performance
- The `@submission_date` parameter is used during ETL execution
- Table is clustered by `client_id` and `sample_id` for improved query performance
- Average row size varies significantly based on array fields (active_addons, experiments, search_counts)

---

**Generated with Enhanced Documentation Process**
**Last Updated**: 2024 (Schema v6)
**Maintained by**: Mozilla Data Platform Team
