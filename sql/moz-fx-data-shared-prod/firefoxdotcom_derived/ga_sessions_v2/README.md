# Google Analytics Sessions V2

## Overview

This table contains session-level aggregations from Google Analytics 4 events for Firefox.com. Each row represents a unique session identified by the composite primary key (ga_client_id, ga_session_id). The table is incrementally updated daily, reprocessing sessions with events in the last 3 days to ensure data completeness.

## Key Features

- **Partitioning**: Time-partitioned by session_date (day-level, 775 days retention)
- **Clustering**: Optimized queries on ga_client_id and country
- **Update Strategy**: MERGE operation that upserts sessions with events from the past 3 days
- **Source**: moz-fx-data-marketing-prod.analytics_489412379.events_*

## Data Model

The ETL consolidates session attributes from two main sources:

1. **Session Start Event**: Captures device properties, location, traffic source, and advertising attribution at the moment of session_start
2. **Event Parameters**: Aggregates campaign, source, medium, content, and experiment information from all events within the session

## Downstream Analysis Suggestions

### 1. Campaign Performance & Attribution Analysis

Analyze the effectiveness of marketing campaigns by comparing attribution sources and conversion metrics:

```sql
SELECT 
  manual_source,
  manual_medium,
  manual_campaign_name,
  COUNT(DISTINCT ga_client_id) AS unique_users,
  COUNT(DISTINCT ga_session_id) AS total_sessions,
  SUM(firefox_desktop_downloads) AS total_downloads,
  SAFE_DIVIDE(SUM(firefox_desktop_downloads), COUNT(DISTINCT ga_session_id)) AS download_rate,
  AVG(time_on_site) AS avg_session_duration,
  AVG(pageviews) AS avg_pageviews
FROM `moz-fx-data-shared-prod.firefoxdotcom_derived.ga_sessions_v2`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND manual_campaign_name IS NOT NULL
GROUP BY 1, 2, 3
ORDER BY total_downloads DESC;
```

### 2. User Journey & Conversion Funnel Analysis

Track user progression from first session to download conversion across sessions:

```sql
WITH user_sessions AS (
  SELECT 
    ga_client_id,
    session_date,
    is_first_session,
    session_number,
    had_download_event,
    firefox_desktop_downloads,
    landing_screen,
    device_category,
    country
  FROM `moz-fx-data-shared-prod.firefoxdotcom_derived.ga_sessions_v2`
  WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
)
SELECT 
  is_first_session,
  landing_screen,
  device_category,
  COUNT(DISTINCT ga_client_id) AS unique_users,
  COUNTIF(had_download_event) AS sessions_with_download,
  SAFE_DIVIDE(COUNTIF(had_download_event), COUNT(*)) AS conversion_rate
FROM user_sessions
GROUP BY 1, 2, 3
HAVING unique_users >= 100
ORDER BY unique_users DESC;
```

### 3. Cross-Device & Geographic Engagement Analysis

Compare user behavior and engagement patterns across device types and geographic regions:

```sql
SELECT 
  device_category,
  country,
  browser,
  COUNT(DISTINCT ga_client_id) AS unique_users,
  COUNT(DISTINCT ga_session_id) AS total_sessions,
  AVG(pageviews) AS avg_pageviews_per_session,
  AVG(time_on_site) AS avg_time_on_site,
  COUNTIF(had_download_event) AS sessions_with_downloads,
  SAFE_DIVIDE(COUNTIF(had_download_event), COUNT(*)) AS download_conversion_rate
FROM `moz-fx-data-shared-prod.firefoxdotcom_derived.ga_sessions_v2`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND device_category IS NOT NULL
GROUP BY 1, 2, 3
ORDER BY unique_users DESC;
```

### 4. Experiment Impact & A/B Testing Analysis

Evaluate the performance of experiments by comparing key metrics across experiment branches:

```sql
WITH experiment_sessions AS (
  SELECT 
    first_experiment_id_from_event_params AS experiment_id,
    first_experiment_branch_from_event_params AS experiment_branch,
    COUNT(DISTINCT ga_client_id) AS unique_users,
    COUNT(DISTINCT ga_session_id) AS total_sessions,
    AVG(pageviews) AS avg_pageviews,
    AVG(time_on_site) AS avg_time_on_site,
    COUNTIF(had_download_event) AS sessions_with_download,
    SUM(firefox_desktop_downloads) AS total_downloads
  FROM `moz-fx-data-shared-prod.firefoxdotcom_derived.ga_sessions_v2`
  WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
    AND first_experiment_id_from_event_params IS NOT NULL
  GROUP BY 1, 2
)
SELECT 
  experiment_id,
  experiment_branch,
  unique_users,
  total_sessions,
  ROUND(avg_pageviews, 2) AS avg_pageviews,
  ROUND(avg_time_on_site, 2) AS avg_time_on_site_seconds,
  total_downloads,
  SAFE_DIVIDE(sessions_with_download, total_sessions) AS conversion_rate
FROM experiment_sessions
WHERE unique_users >= 100
ORDER BY experiment_id, experiment_branch;
```

## Additional Notes

- Sessions without a ga_session_id are excluded from this table
- The table uses America/Los_Angeles timezone for timestamp conversions
- Download events include: product_download, firefox_download, firefox_mobile_download, focus_download, and klar_download
- Google Click IDs (gclid) enable joining with Google Ads performance data
- Stub session IDs can be joined with dl_ga_triplets to retrieve download tokens