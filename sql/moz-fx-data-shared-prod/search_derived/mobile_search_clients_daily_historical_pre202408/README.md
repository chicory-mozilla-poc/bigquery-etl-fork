# Mobile Search Clients Daily Historical (Pre-August 2024)

## Overview

The `mobile_search_clients_daily_historical_pre202408` table contains client-level daily search metrics for Mozilla's mobile Firefox products (Firefox for Android, Firefox for iOS, Focus, etc.) for the historical period up to and including **July 31, 2024**.

This table represents a critical snapshot of search behavior during the transition period when Mozilla moved from collecting metrics via the **metrics ping** to the **baseline ping**. The cutoff date of July 31, 2024, marks the end of the metrics ping era, with all subsequent data being collected through baseline ping and stored in the current operational tables.

### Key Characteristics

- **Data Period**: All dates up to and including 2024-07-31
- **Granularity**: One row per client per day per search engine per source combination
- **Table Type**: Client-level aggregated metrics
- **Partition Field**: `submission_date`
- **Data Retention**: 775 days (approximately 2 years)
- **Source Ping Type**: Metrics ping (historical method)

## Purpose

This table serves several critical functions:

1. **Historical Analysis**: Enables longitudinal studies comparing search behavior across the metrics-to-baseline ping transition
2. **Baseline Comparison**: Provides a reference dataset for validating the new baseline ping methodology
3. **Trend Analysis**: Supports multi-year search trend analysis when combined with current tables
4. **Data Continuity**: Ensures no data loss during the migration period
5. **Partner Reporting**: Maintains historical search partner revenue data for reconciliation

## Schema

The table contains 33 columns organized into the following categories:

### Identifiers & Temporal Dimensions
- `submission_date`: Partition key for the data collection date (≤ 2024-07-31)
- `client_id`: Unique Glean client identifier
- `sample_id`: Random sampling identifier (0-99)

### Application Metadata
- `app_name`: Raw mobile app identifier (Fenix, Firefox iOS, Focus)
- `normalized_app_name`: Standardized app name for cross-product analysis
- `app_version`: Application version string
- `channel`: Release channel (release, beta, nightly)
- `distribution_id`: Partner distribution identifier

### Search Dimensions
- `engine`: Raw search engine identifier
- `normalized_engine`: Standardized search engine name
- `source`: Search initiation source (urlbar, searchbar, widget, etc.)
- `default_search_engine`: User's configured default search provider
- `default_search_engine_submission_url`: Full search URL template with partner codes

### Search Metrics (Core KPIs)
- `search_count`: Total searches performed
- `organic`: Unmonetized searches without partner tags
- `tagged_sap`: Searches from Mozilla partner access points (revenue-generating)
- `tagged_follow_on`: Subsequent searches after initial SAP search (revenue-generating)
- `unknown`: Unclassified searches

### Advertising Metrics
- `ad_click`: Total ad clicks across all search types
- `search_with_ads`: Searches displaying advertisements
- `ad_click_organic`: Ad clicks on organic searches
- `search_with_ads_organic`: Organic searches with ads displayed

### Geographic & Localization
- `country`: ISO country code from IP geolocation
- `locale`: Application locale setting (e.g., en-US, de-DE)

### Platform Information
- `os`: Operating system (Android, iOS)
- `os_version`: Full OS version string
- `os_version_major`: Extracted major version number
- `os_version_minor`: Extracted minor version number

### User Context
- `profile_creation_date`: Profile creation timestamp (days since epoch)
- `profile_age_in_days`: Calculated profile age
- `total_uri_count`: Daily page load count for activity context

### Experimentation
- `experiments`: Array of active experiment enrollments (key-value pairs)

## Data Sources

### Primary Source
- **Table**: `moz-fx-data-shared-prod.search_derived.mobile_search_clients_daily_v1`
- **Filter**: `submission_date <= '2024-07-31'`
- **Ping Type**: Metrics ping (Glean)

### Upstream Dependencies
The source table aggregates data from:
- Mobile metrics ping telemetry
- Glean client_info metadata
- Search telemetry events
- GeoIP lookup services

## Update Schedule

**Status**: Static historical table (no ongoing updates)

This table is **frozen** as of the August 1, 2024 cutoff date. It does not receive updates and serves purely as an archival dataset. For current mobile search data (August 2024 onwards), query the baseline ping-based tables.

### Partitioning
- **Type**: Daily partitioning by `submission_date`
- **Required**: Partition filter is required for all queries (performance optimization)
- **Expiration**: Partitions expire after 775 days

## Usage Examples

### Example 1: Daily Search Volume by App
```sql
SELECT
  submission_date,
  normalized_app_name,
  COUNT(DISTINCT client_id) AS active_clients,
  SUM(search_count) AS total_searches,
  SUM(tagged_sap) AS monetized_searches
FROM
  `moz-fx-data-shared-prod.search_derived.mobile_search_clients_daily_historical_pre202408`
WHERE
  submission_date BETWEEN '2024-07-01' AND '2024-07-31'
GROUP BY
  submission_date,
  normalized_app_name
ORDER BY
  submission_date DESC;
```

### Example 2: Search Engine Market Share
```sql
SELECT
  normalized_engine,
  COUNT(DISTINCT client_id) AS users,
  SUM(search_count) AS searches,
  ROUND(SUM(search_count) * 100.0 / SUM(SUM(search_count)) OVER (), 2) AS market_share_pct
FROM
  `moz-fx-data-shared-prod.search_derived.mobile_search_clients_daily_historical_pre202408`
WHERE
  submission_date BETWEEN '2024-06-01' AND '2024-07-31'
  AND normalized_engine IS NOT NULL
GROUP BY
  normalized_engine
ORDER BY
  searches DESC;
```

### Example 3: Geographic Distribution Analysis
```sql
SELECT
  country,
  COUNT(DISTINCT client_id) AS active_clients,
  SUM(tagged_sap + tagged_follow_on) AS monetized_searches,
  ROUND(SUM(ad_click) * 100.0 / NULLIF(SUM(search_with_ads), 0), 2) AS ad_ctr_pct
FROM
  `moz-fx-data-shared-prod.search_derived.mobile_search_clients_daily_historical_pre202408`
WHERE
  submission_date BETWEEN '2024-07-01' AND '2024-07-31'
  AND country IS NOT NULL
GROUP BY
  country
HAVING
  active_clients >= 100
ORDER BY
  monetized_searches DESC
LIMIT 20;
```

### Example 4: Cohort Analysis by Profile Age
```sql
SELECT
  CASE
    WHEN profile_age_in_days <= 7 THEN '0-7 days'
    WHEN profile_age_in_days <= 30 THEN '8-30 days'
    WHEN profile_age_in_days <= 90 THEN '31-90 days'
    WHEN profile_age_in_days <= 365 THEN '91-365 days'
    ELSE '365+ days'
  END AS age_cohort,
  COUNT(DISTINCT client_id) AS clients,
  AVG(search_count) AS avg_searches_per_client,
  AVG(total_uri_count) AS avg_page_loads
FROM
  `moz-fx-data-shared-prod.search_derived.mobile_search_clients_daily_historical_pre202408`
WHERE
  submission_date = '2024-07-31'
  AND profile_age_in_days IS NOT NULL
GROUP BY
  age_cohort
ORDER BY
  MIN(profile_age_in_days);
```

### Example 5: Experiment Impact Analysis
```sql
WITH experiment_enrollments AS (
  SELECT
    client_id,
    submission_date,
    search_count,
    exp.value AS experiment_branch
  FROM
    `moz-fx-data-shared-prod.search_derived.mobile_search_clients_daily_historical_pre202408`,
    UNNEST(experiments) AS exp
  WHERE
    submission_date BETWEEN '2024-07-01' AND '2024-07-31'
    AND exp.key = 'experiment-search-ui-2024'
)
SELECT
  experiment_branch,
  COUNT(DISTINCT client_id) AS enrolled_clients,
  AVG(search_count) AS avg_daily_searches,
  STDDEV(search_count) AS search_stddev
FROM
  experiment_enrollments
GROUP BY
  experiment_branch
ORDER BY
  experiment_branch;
```

## Related Tables

### Current Mobile Search Tables
- **`mobile_search_clients_daily_v1`**: Current operational table (all dates, including post-July 2024 baseline ping data)
- **`mobile_search_aggregates_v1`**: Aggregated mobile search metrics without client_id
- **`search_clients_engines_sources_daily`**: Combined desktop + mobile search data

### Desktop Search Tables
- **`search_clients_daily_v8`**: Desktop Firefox search metrics
- **`search_clients_engines_sources_daily`**: Unified desktop search with engine/source breakdown

### Experimental/Comparative Tables
- **`mobile_search_clients_daily_v2`**: May contain baseline ping implementation (verify schema)

## Important Notes

### Data Quality Considerations
1. **Metrics Ping Limitations**: This data was collected via metrics ping, which had known limitations around:
   - Delayed submission timing
   - Potential data loss on app termination
   - Different aggregation windows compared to baseline ping

2. **Transition Artifacts**: Dates near the July 31, 2024 cutoff may show anomalies due to the migration process

3. **Sampling**: Use `sample_id` field for consistent random sampling (each sample_id represents ~1% of clients)

### Performance Best Practices
1. **Always** include `submission_date` in WHERE clause (partition filter required)
2. Use `sample_id` for large-scale exploratory analysis: `WHERE sample_id = 0` (1% sample)
3. Filter on indexed fields (`client_id`, `country`, `normalized_engine`) when possible
4. Consider pre-aggregating at weekly/monthly level for long-term trend analysis

### Privacy & Data Governance
- Client-level data: Requires appropriate data access permissions
- Contains potentially sensitive search behavior patterns
- Must comply with Mozilla's data retention and deletion policies
- Be mindful of re-identification risks when combining with other datasets

## Owners

- **Primary**: akommasani@mozilla.com
- **Secondary**: mbowerman@mozilla.com

For questions about data quality, schema changes, or access issues, contact the table owners or the Mozilla Data Team (#data-help on Slack).

## Changelog

- **2024-08**: Table created to preserve historical metrics ping data during transition to baseline ping methodology
- **Cutoff Date**: 2024-07-31 (inclusive) - all data in this table is from metrics ping era
