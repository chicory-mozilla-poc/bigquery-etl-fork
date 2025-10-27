# Google Analytics Sessions V3

## Overview

The `ga_sessions_v3` table contains session-level data from Google Analytics 4 for mozilla.org traffic. Each row represents a unique session identified by the composite key (`ga_client_id`, `ga_session_id`). The table is incrementally updated using a MERGE operation that processes events from the last 3 days to capture both new sessions and updates to existing sessions.

## Data Source

This table is derived from the Google Analytics 4 events data in `moz-fx-data-marketing-prod.analytics_313696158.events_*`. The ETL process aggregates event-level data into session-level metrics and attributes.

## Key Features

- **Incremental Updates**: Processes events from 3 days prior to submission date to capture late-arriving data
- **Session Attributes**: Captures device, geographic, and browser information at session start
- **Attribution Tracking**: Includes both manual UTM parameters and Google Ads attribution data
- **Download Tracking**: Tracks Firefox downloads with detailed install target information
- **Experiment Support**: Captures A/B test experiment IDs and branches
- **Cross-Channel Data**: Includes Google Ads cross-channel campaign attribution

## Table Structure

- **Primary Key**: Composite key of `ga_client_id` and `ga_session_id`
- **Partitioning**: Day-partitioned on `session_date` with 775-day expiration
- **Clustering**: Clustered by `ga_client_id` and `country` for query optimization
- **Total Columns**: 62 columns covering session metrics, attribution, and user attributes

## ETL Logic

The MERGE script performs the following operations:

1. **Identifies Active Sessions**: Finds all ga_client_id/ga_session_id combinations with events in the last 3 days
2. **Session Start Attributes**: Extracts device, location, and attribution data from session_start events
3. **Event Aggregation**: Calculates pageviews, time on site, and download metrics across all session events
4. **Attribution Processing**: Collects first and distinct values from event_params for campaigns, sources, mediums, etc.
5. **MERGE Operation**: Inserts new sessions or updates existing sessions with latest data

## Downstream Analysis Suggestions

### 1. Campaign Performance Analysis
```sql
-- Analyze download conversion rates by campaign
SELECT 
  manual_campaign_name,
  COUNT(DISTINCT ga_session_id) as total_sessions,
  COUNTIF(had_download_event) as sessions_with_downloads,
  SAFE_DIVIDE(COUNTIF(had_download_event), COUNT(DISTINCT ga_session_id)) as download_rate,
  SUM(firefox_desktop_downloads) as total_downloads
FROM `moz-fx-data-shared-prod.mozilla_org_derived.ga_sessions_v3`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND manual_campaign_name IS NOT NULL
GROUP BY manual_campaign_name
ORDER BY total_sessions DESC;
```

### 2. Geographic User Engagement
```sql
-- Analyze session engagement metrics by country
SELECT 
  country,
  COUNT(DISTINCT ga_client_id) as unique_users,
  COUNT(DISTINCT ga_session_id) as total_sessions,
  AVG(time_on_site) as avg_session_duration_seconds,
  AVG(pageviews) as avg_pageviews_per_session,
  COUNTIF(is_first_session) as first_time_sessions
FROM `moz-fx-data-shared-prod.mozilla_org_derived.ga_sessions_v3`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
GROUP BY country
HAVING total_sessions >= 100
ORDER BY total_sessions DESC
LIMIT 20;
```

### 3. Cross-Channel Attribution
```sql
-- Compare performance across traffic channels
SELECT 
  COALESCE(ad_crosschannel_default_channel_group, 'Direct/Other') as channel_group,
  manual_medium,
  COUNT(DISTINCT ga_session_id) as sessions,
  AVG(pageviews) as avg_pageviews,
  COUNTIF(had_download_event) as download_sessions,
  SAFE_DIVIDE(SUM(firefox_desktop_downloads), COUNT(DISTINCT ga_session_id)) as downloads_per_session
FROM `moz-fx-data-shared-prod.mozilla_org_derived.ga_sessions_v3`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY channel_group, manual_medium
ORDER BY sessions DESC;
```

### 4. Experiment Performance Tracking
```sql
-- Analyze download rates by experiment variants
WITH experiment_sessions AS (
  SELECT 
    first_experiment_id_from_event_params as experiment_id,
    first_experiment_branch_from_event_params as variant,
    COUNT(DISTINCT ga_session_id) as sessions,
    COUNTIF(had_download_event) as download_sessions,
    SUM(firefox_desktop_downloads) as total_downloads,
    AVG(time_on_site) as avg_engagement_seconds
  FROM `moz-fx-data-shared-prod.mozilla_org_derived.ga_sessions_v3`
  WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 14 DAY)
    AND first_experiment_id_from_event_params IS NOT NULL
  GROUP BY experiment_id, variant
)
SELECT 
  experiment_id,
  variant,
  sessions,
  download_sessions,
  SAFE_DIVIDE(download_sessions, sessions) as conversion_rate,
  SAFE_DIVIDE(total_downloads, sessions) as downloads_per_session,
  avg_engagement_seconds
FROM experiment_sessions
ORDER BY experiment_id, variant;
```

## Data Quality Notes

- Sessions without a `ga_session_id` are excluded from the table
- Session attributes default to first event values when session_start event is unavailable
- The table maintains historical sessions but partitions expire after 775 days
- Late-arriving events are captured through the 3-day lookback window

## Access

- **Owner**: kwindau@mozilla.com
- **Workgroup Access**: external-ads-datafusion, external-ads-dataproc (roles/bigquery.dataViewer)
- **DAG**: bqetl_google_analytics_derived_ga4
