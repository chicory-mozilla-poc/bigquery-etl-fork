# Active Users Aggregates Device v1

## Overview

This table provides aggregated metrics of daily, weekly, and monthly active users, along with new profiles and search counts, segmented by device-level attributes. It is optimized for device-specific analysis by separating device_model from other aggregations, improving query performance for analysts examining user behavior across different devices.

## Data Source

**Source Table:** `moz-fx-data-shared-prod.telemetry.unified_metrics`

**Update Frequency:** Daily (incremental)

**Partitioning:** Time-partitioned by `submission_date` (day-level)

**Clustering:** `country`, `app_name`, `attribution_medium`, `channel`

## Key Metrics

- **User Activity Metrics:** DAU, WAU, MAU, QDAU (Desktop)
- **Growth Metrics:** New profiles
- **Engagement Metrics:** Active hours, URI count
- **Search Metrics:** Search count, organic search count, search with ads, ad clicks
- **Attribution Tracking:** Attribution source and medium with attributed flag

## Schema Dimensions

The table aggregates metrics across the following dimensions:
- Device characteristics (device_model, os, os_version)
- Application attributes (app_name, app_version, channel)
- Geographic location (country)
- Attribution data (attribution_source, attribution_medium)
- User cohort (first_seen_year, is_default_browser)

## Downstream Analysis Suggestions

### 1. Device Performance Analysis
**Objective:** Compare user engagement and retention across different device models
```sql
SELECT
  device_model,
  os,
  AVG(dau) as avg_daily_users,
  AVG(active_hours/NULLIF(dau,0)) as avg_hours_per_user,
  AVG(uri_count/NULLIF(dau,0)) as avg_pages_per_user
FROM `moz-fx-data-shared-prod.telemetry_derived.active_users_aggregates_device_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY device_model, os
ORDER BY avg_daily_users DESC
```

### 2. Attribution Channel Effectiveness
**Objective:** Measure user acquisition and engagement quality by attribution source
```sql
SELECT
  attribution_source,
  attribution_medium,
  SUM(new_profiles) as total_new_users,
  SUM(dau)/COUNT(DISTINCT submission_date) as avg_daily_active,
  SUM(search_count)/NULLIF(SUM(dau),0) as searches_per_user
FROM `moz-fx-data-shared-prod.telemetry_derived.active_users_aggregates_device_v1`
WHERE submission_date BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY) AND CURRENT_DATE()
  AND attributed = TRUE
GROUP BY attribution_source, attribution_medium
ORDER BY total_new_users DESC
```

### 3. Geographic Market Share Analysis
**Objective:** Identify growth opportunities and market penetration by country
```sql
SELECT
  country,
  app_name,
  channel,
  SUM(mau) as monthly_users,
  SUM(new_profiles) as new_users,
  SUM(new_profiles)/NULLIF(SUM(mau),0) as growth_rate
FROM `moz-fx-data-shared-prod.telemetry_derived.active_users_aggregates_device_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 28 DAY)
GROUP BY country, app_name, channel
HAVING SUM(mau) > 1000
ORDER BY monthly_users DESC
```

### 4. OS Version Adoption and Compatibility
**Objective:** Monitor OS version distribution to inform compatibility decisions
```sql
SELECT
  os,
  os_version_major,
  os_version_minor,
  SUM(dau) as daily_users,
  SUM(dau) / SUM(SUM(dau)) OVER (PARTITION BY os) as version_share
FROM `moz-fx-data-shared-prod.telemetry_derived.active_users_aggregates_device_v1`
WHERE submission_date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
GROUP BY os, os_version_major, os_version_minor
QUALIFY version_share > 0.01
ORDER BY os, daily_users DESC
```

## Owners

- **Primary Owner:** lvargas@mozilla.com

## Notes

- This table requires a partition filter on `submission_date` for all queries
- QDAU (Qualified DAU) is only calculated for Firefox Desktop and requires active_hours > 0 and uri_count > 0
- The `attributed` flag is a computed boolean indicating presence of any attribution data
- Device model is particularly useful for mobile analysis where models vary significantly
