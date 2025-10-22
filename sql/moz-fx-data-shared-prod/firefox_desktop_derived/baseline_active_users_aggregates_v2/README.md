# Baseline Active Users Aggregates V2

## Overview
The `baseline_active_users_aggregates_v2` table provides aggregated active user metrics for Firefox Desktop, built from baseline ping data. This table aggregates user counts across multiple dimensions including geographic location, platform details, browser versions, and user acquisition channels.

## Table Details
- **Dataset**: `firefox_desktop_derived`
- **Source Table**: `firefox_desktop.baseline_active_users`
- **Partition**: Daily partitioning by `submission_date`
- **Clustering**: Optimized by `app_name`, `channel`, and `country`
- **Update Frequency**: Daily incremental updates

## Owners
- kwindau@mozilla.com
- ago@mozilla.com

## Data Structure

### Dimensional Attributes
The table includes rich dimensional data for slicing user metrics:
- **Temporal**: submission_date, first_seen_year
- **Product**: channel, app_name, app_version details
- **Geographic**: country, city, locale
- **Platform**: os, os_version, windows_build_number
- **User Behavior**: activity_segment, is_default_browser
- **Attribution**: attribution_medium, attribution_source, distribution_id

### User Metrics
The table provides six key user count metrics:
- **daily_users**: Users who sent pings on the given date
- **weekly_users**: Users who sent pings within the past 7 days
- **monthly_users**: Users who sent pings within the past 28 days
- **dau**: Daily Active Users based on activity criteria
- **wau**: Weekly Active Users based on activity criteria
- **mau**: Monthly Active Users based on activity criteria

## ETL Logic
The table aggregates user-level data from `baseline_active_users` using `COUNTIF` to sum boolean flags (is_daily_user, is_weekly_user, etc.) across all dimensional combinations. Each row represents a unique combination of dimensions with corresponding user counts.

## Downstream Analysis Suggestions

### 1. Cross-Platform User Growth Analysis
Analyze user growth trends across different operating systems and versions to identify platform-specific adoption patterns:
```sql
SELECT 
  submission_date,
  os,
  os_version_major,
  SUM(dau) as total_dau,
  SUM(wau) as total_wau,
  SUM(mau) as total_mau
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.baseline_active_users_aggregates_v2`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY 1, 2, 3
ORDER BY submission_date DESC, total_mau DESC
```

### 2. Channel Performance and User Retention
Compare user retention patterns across release channels (release, beta, nightly) to understand channel health:
```sql
SELECT 
  channel,
  activity_segment,
  SUM(dau) as daily_active,
  SUM(wau) as weekly_active,
  SUM(mau) as monthly_active,
  SAFE_DIVIDE(SUM(dau), SUM(mau)) as dau_mau_ratio
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.baseline_active_users_aggregates_v2`
WHERE submission_date = CURRENT_DATE() - 1
GROUP BY 1, 2
ORDER BY monthly_active DESC
```

### 3. Geographic Market Penetration Analysis
Identify markets where Firefox is set as the default browser to prioritize growth initiatives:
```sql
SELECT 
  country,
  is_default_browser,
  SUM(mau) as monthly_users,
  ROUND(100.0 * SUM(mau) / SUM(SUM(mau)) OVER (PARTITION BY country), 2) as pct_of_country
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.baseline_active_users_aggregates_v2`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 28 DAY)
  AND country != '??'
GROUP BY 1, 2
HAVING SUM(mau) > 1000
ORDER BY country, monthly_users DESC
```

### 4. Attribution Source Effectiveness
Evaluate which marketing channels drive the most engaged users by analyzing attribution data:
```sql
SELECT 
  attribution_source,
  attribution_medium,
  first_seen_year,
  SUM(dau) as daily_active,
  SUM(mau) as monthly_active,
  SAFE_DIVIDE(SUM(dau), SUM(mau)) as engagement_rate
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.baseline_active_users_aggregates_v2`
WHERE submission_date = CURRENT_DATE() - 1
  AND attribution_source IS NOT NULL
GROUP BY 1, 2, 3
ORDER BY monthly_active DESC
LIMIT 20
```

## Usage Notes
- Always include a partition filter on `submission_date` for query performance
- Use clustering columns (app_name, channel, country) in WHERE clauses for optimization
- The table is incrementally updated daily via the `bqetl_analytics_aggregations` DAG
- Unknown country values are represented as '??' for privacy compliance