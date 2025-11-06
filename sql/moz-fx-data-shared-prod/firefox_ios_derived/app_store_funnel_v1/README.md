# Firefox iOS App Store Funnel

## Overview

The `app_store_funnel_v1` table provides a comprehensive view of the Firefox iOS acquisition funnel, combining Apple App Store metrics with Firefox profile creation data. This derived dataset enables analysis of user journey from app discovery through installation and activation.

## Data Sources

- **Apple App Store Reports**: Impression and download data from territory and source type reports
- **Firefox iOS Telemetry**: New profile creation data from the `firefox_ios.new_profiles` table
- **Static Reference Data**: Country name normalization from `static.country_names_v1`

## Key Features

- **7-Day Lag**: Data runs with a 7-day delay to account for Apple's reporting latency
- **Daily Partitioning**: Partitioned by `submission_date` for efficient querying
- **Country-Level Aggregation**: Metrics grouped by normalized country codes
- **Release Channel Focus**: Includes only release channel profiles (excludes beta/nightly)
- **Historical Continuity**: Combines legacy and current Apple reporting formats (transition date: 2024-01-01)

## ETL Logic

The query performs the following transformations:

1. **Historical Store Data** (pre-2024): Aggregates impressions and downloads from `app_store.firefox_app_store_territory_source_type_report`
2. **Current Store Data** (2024+): Aggregates from Fivetran-sourced `firefox_app_store_v2_apple_store.apple_store__territory_report`
3. **Data Filtering**: Excludes institutional purchases and non-standard source types
4. **Country Normalization**: Converts country names to ISO codes
5. **Profile Matching**: Joins with Firefox telemetry to connect downloads to profile activations

## Column Descriptions

| Column | Type | Description |
|--------|------|-------------|
| `submission_date` | DATE | Partition field representing data processing date |
| `first_seen_date` | DATE | Date of app store activity (submission_date - 7 days) |
| `country` | STRING | ISO country code where activity occurred |
| `impressions` | INTEGER | Unique device impressions in Apple App Store |
| `total_downloads` | INTEGER | Total app downloads (first-time + redownloads) |
| `first_time_downloads` | INTEGER | First-time installations |
| `redownloads` | INTEGER | Reinstallations by previous users |
| `new_profiles` | INTEGER | Firefox iOS profiles created on first_seen_date |

## Downstream Analysis Suggestions

### 1. Funnel Conversion Analysis
Calculate conversion rates at each stage of the user acquisition funnel:
```sql
SELECT
  first_seen_date,
  country,
  SUM(impressions) AS total_impressions,
  SUM(total_downloads) AS total_downloads,
  SUM(new_profiles) AS activated_profiles,
  SAFE_DIVIDE(SUM(total_downloads), SUM(impressions)) AS impression_to_download_rate,
  SAFE_DIVIDE(SUM(new_profiles), SUM(total_downloads)) AS download_to_activation_rate
FROM `moz-fx-data-shared-prod.firefox_ios_derived.app_store_funnel_v1`
WHERE first_seen_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY first_seen_date, country
ORDER BY first_seen_date DESC;
```

### 2. Geographic Performance Comparison
Identify high-performing markets and expansion opportunities:
```sql
SELECT
  country,
  AVG(SAFE_DIVIDE(new_profiles, total_downloads)) AS avg_activation_rate,
  SUM(total_downloads) AS total_downloads,
  SUM(new_profiles) AS total_activations
FROM `moz-fx-data-shared-prod.firefox_ios_derived.app_store_funnel_v1`
WHERE first_seen_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY country
HAVING total_downloads > 100
ORDER BY avg_activation_rate DESC
LIMIT 20;
```

### 3. User Re-engagement Trends
Analyze the proportion of returning users versus new acquisitions:
```sql
SELECT
  DATE_TRUNC(first_seen_date, WEEK) AS week,
  SUM(first_time_downloads) AS new_users,
  SUM(redownloads) AS returning_users,
  SAFE_DIVIDE(SUM(redownloads), SUM(total_downloads)) AS redownload_rate
FROM `moz-fx-data-shared-prod.firefox_ios_derived.app_store_funnel_v1`
WHERE first_seen_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
GROUP BY week
ORDER BY week DESC;
```

### 4. Activation Gap Analysis
Identify downloads that don't result in profile creation (potential onboarding issues):
```sql
SELECT
  first_seen_date,
  country,
  SUM(total_downloads) AS downloads,
  SUM(new_profiles) AS profiles_created,
  SUM(total_downloads) - SUM(new_profiles) AS activation_gap,
  SAFE_DIVIDE(SUM(total_downloads) - SUM(new_profiles), SUM(total_downloads)) AS gap_percentage
FROM `moz-fx-data-shared-prod.firefox_ios_derived.app_store_funnel_v1`
WHERE first_seen_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 60 DAY)
  AND total_downloads > 0
GROUP BY first_seen_date, country
HAVING gap_percentage > 0.3
ORDER BY activation_gap DESC
LIMIT 50;
```

## Scheduling & Maintenance

- **DAG**: `bqetl_firefox_ios`
- **Cadence**: Daily
- **Dependencies**: Requires prior completion of Apple App Store data ingestion
- **Monitoring**: Enabled via standard BigQuery ETL monitoring
- **Owner**: kik@mozilla.com

## Notes & Caveats

- Apple reports are based on Pacific Time (PT) without timezone conversion
- Institutional purchases are excluded from all metrics
- Historical data (pre-2024) uses different source tables than current data
- Profile creation may lag download by several days (not captured in this table)
- Null country codes ('??') indicate unknown geolocation
