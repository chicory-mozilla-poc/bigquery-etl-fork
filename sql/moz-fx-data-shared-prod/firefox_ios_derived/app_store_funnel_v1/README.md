# App Store Funnel v1

## Overview

The `app_store_funnel_v1` table provides a comprehensive view of the Firefox iOS app acquisition funnel from Apple App Store impressions through to profile creation. This derived dataset combines Apple App Store metrics with Firefox telemetry data to enable end-to-end funnel analysis, running with a 7-day lag to accommodate App Store data availability.

## Data Sources

The table integrates data from multiple sources:
- **Historical App Store Data** (pre-2024): `firefox_app_store_territory_source_type_report` and `firefox_downloads_territory_source_type_report`
- **Current App Store Data** (2024+): `firefox_app_store_v2_apple_store.apple_store__territory_report` via Fivetran
- **Profile Data**: `firefox_ios.new_profiles` for release channel profile creation metrics
- **Country Normalization**: `static.country_names_v1` for standardized country codes

## Schema

| Column | Type | Description |
|--------|------|-------------|
| `submission_date` | DATE | The date when the telemetry ping is received on the server side |
| `first_seen_date` | DATE | Date when the user was first observed, offset by 7 days from submission_date |
| `country` | STRING | Name of the country in which the activity took place |
| `impressions` | INTEGER | Unique device impressions in the Apple App Store |
| `total_downloads` | INTEGER | Total downloads including first-time and redownloads |
| `first_time_downloads` | INTEGER | Initial downloads from new users |
| `redownloads` | INTEGER | Downloads from returning users |
| `new_profiles` | INTEGER | New Firefox iOS release channel profiles created |

## Key Characteristics

- **Partition Field**: `submission_date` (day-level time partitioning)
- **Granularity**: Daily metrics aggregated by country
- **Update Schedule**: Daily with 7-day lag
- **Data Lag Rationale**: Apple App Store data becomes available several days after the event date
- **App Filter**: Restricted to Firefox iOS (app_id = 989804926)
- **Exclusions**: Institutional purchases excluded from all metrics
- **Impression Logic**: Web referrers and app referrers excluded from impression counts to align with Apple's current methodology

## Downstream Analysis Suggestions

### 1. Conversion Funnel Analysis
Analyze the conversion rates at each stage of the user acquisition funnel:
```sql
SELECT
  country,
  SUM(impressions) AS total_impressions,
  SUM(total_downloads) AS total_downloads,
  SUM(new_profiles) AS total_profiles,
  SAFE_DIVIDE(SUM(total_downloads), SUM(impressions)) AS impression_to_download_rate,
  SAFE_DIVIDE(SUM(new_profiles), SUM(total_downloads)) AS download_to_profile_rate,
  SAFE_DIVIDE(SUM(new_profiles), SUM(impressions)) AS impression_to_profile_rate
FROM `moz-fx-data-shared-prod.firefox_ios_derived.app_store_funnel_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY country
ORDER BY total_impressions DESC
```

### 2. User Retention Indicator
Compare first-time downloads vs redownloads to understand user retention patterns:
```sql
SELECT
  first_seen_date,
  country,
  first_time_downloads,
  redownloads,
  SAFE_DIVIDE(redownloads, first_time_downloads) AS redownload_ratio
FROM `moz-fx-data-shared-prod.firefox_ios_derived.app_store_funnel_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  AND first_time_downloads > 0
ORDER BY first_seen_date DESC
```

### 3. Geographic Performance Trends
Identify high-performing markets and track performance changes over time:
```sql
SELECT
  country,
  DATE_TRUNC(first_seen_date, MONTH) AS month,
  AVG(SAFE_DIVIDE(total_downloads, impressions)) AS avg_download_rate,
  AVG(SAFE_DIVIDE(new_profiles, total_downloads)) AS avg_activation_rate,
  SUM(total_downloads) AS total_monthly_downloads
FROM `moz-fx-data-shared-prod.firefox_ios_derived.app_store_funnel_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
GROUP BY country, month
HAVING SUM(impressions) > 1000
ORDER BY month DESC, total_monthly_downloads DESC
```

### 4. Weekly Cohort Performance
Track weekly cohorts to identify seasonality and campaign impact:
```sql
SELECT
  DATE_TRUNC(first_seen_date, WEEK) AS week,
  SUM(impressions) AS weekly_impressions,
  SUM(first_time_downloads) AS weekly_new_users,
  SUM(new_profiles) AS weekly_profiles,
  SAFE_DIVIDE(SUM(new_profiles), SUM(first_time_downloads)) AS new_user_activation_rate
FROM `moz-fx-data-shared-prod.firefox_ios_derived.app_store_funnel_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY week
ORDER BY week DESC
```

## Notes

- All timestamps are in Pacific Time (PT) as per Apple App Store reporting standards
- The 7-day offset between `submission_date` and `first_seen_date` ensures data completeness
- Country codes are normalized using Mozilla's standard country name mapping
- Profile data is filtered to release channel only for consistency with public app store data
