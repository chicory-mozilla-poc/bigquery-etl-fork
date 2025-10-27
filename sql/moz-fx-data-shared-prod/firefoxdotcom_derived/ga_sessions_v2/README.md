# Google Analytics Sessions V2

## Table Overview

**Table:** `moz-fx-data-shared-prod.firefoxdotcom_derived.ga_sessions_v2`

**Description:** Contains 1 row for each Google Analytics 4 session. The primary key is a composite key (ga_client_id, ga_session_id).

**Owner:** kwindau@mozilla.com

**Scheduling:** Part of the `bqetl_ga4_firefoxdotcom` DAG, runs incrementally with daily partitions on `session_date`.

## Purpose

This table aggregates Google Analytics 4 event data from Firefox.com into session-level metrics. It captures user session behavior including:

- **Session identification and timing** - Unique session IDs, start timestamps, and duration
- **User engagement metrics** - Pageviews, time on site, and download events
- **Traffic attribution** - Campaign sources, mediums, UTM parameters, and Google Click IDs (gclid)
- **Device and location data** - Operating system, browser, device type, and geographic information
- **Advertising attribution** - Google Ads campaign data and cross-channel attribution
- **Experiment tracking** - A/B test experiment IDs and branches

## ETL Process

The table uses a MERGE operation to incrementally update sessions:

1. Identifies all unique GA Client ID / GA Session ID combinations with events in the last 3 days
2. Recalculates session-level information from raw event data
3. Inserts new sessions or overwrites existing sessions with updated data
4. Aggregates attributes from `session_start` events (device properties, location, traffic source)
5. Extracts first and distinct values from nested `event_params` (campaigns, sources, terms, experiments)

**Source:** `moz-fx-data-marketing-prod.analytics_489412379.events_*`

**Partitioning:** Daily partitions on `session_date` with 775-day expiration

**Clustering:** Clustered by `ga_client_id` and `country` for optimized query performance

## Downstream Analysis Suggestions

### 1. Campaign Performance Analysis
Analyze the effectiveness of marketing campaigns by examining:
- Conversion rates (sessions with `had_download_event = TRUE`)
- Attribution sources (`manual_source`, `manual_medium`, `manual_campaign_name`)
- Google Ads performance (`ad_google_campaign`, `ad_crosschannel_campaign`)
- Return on ad spend by correlating `gclid` with cost data

**Example Query:**
```sql
SELECT 
  manual_campaign_name,
  COUNT(DISTINCT ga_client_id) AS unique_users,
  SUM(firefox_desktop_downloads) AS total_downloads,
  AVG(time_on_site) AS avg_time_on_site
FROM `moz-fx-data-shared-prod.firefoxdotcom_derived.ga_sessions_v2`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND manual_campaign_name IS NOT NULL
GROUP BY manual_campaign_name
ORDER BY total_downloads DESC;
```

### 2. User Journey and Cohort Analysis
Track user behavior over time using session sequences:
- First-time vs. returning users (`is_first_session`, `session_number`)
- Landing page effectiveness (`landing_screen`)
- Session progression patterns by cohort
- Time-to-conversion analysis from first session to download event

**Example Query:**
```sql
SELECT 
  session_number,
  COUNT(DISTINCT ga_client_id) AS users,
  SUM(CASE WHEN had_download_event THEN 1 ELSE 0 END) AS sessions_with_downloads,
  AVG(pageviews) AS avg_pageviews
FROM `moz-fx-data-shared-prod.firefoxdotcom_derived.ga_sessions_v2`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY session_number
ORDER BY session_number;
```

### 3. Geographic and Device Segmentation
Segment user behavior by geography and device characteristics:
- Download rates by country and region
- Device category performance (Mobile vs. Desktop vs. Tablet)
- Browser and OS distribution analysis
- Language preferences and localization opportunities

**Example Query:**
```sql
SELECT 
  country,
  device_category,
  COUNT(*) AS total_sessions,
  AVG(pageviews) AS avg_pageviews,
  SUM(firefox_desktop_downloads) AS downloads
FROM `moz-fx-data-shared-prod.firefoxdotcom_derived.ga_sessions_v2`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
GROUP BY country, device_category
HAVING total_sessions > 100
ORDER BY total_sessions DESC;
```

### 4. A/B Testing and Experiment Analysis
Evaluate experiment performance using experiment tracking fields:
- Compare conversion rates across experiment branches
- Analyze engagement metrics by experiment variant
- Track experiment exposure across sessions
- Measure statistical significance of experiment results

**Example Query:**
```sql
SELECT 
  first_experiment_id_from_event_params AS experiment_id,
  first_experiment_branch_from_event_params AS branch,
  COUNT(DISTINCT ga_client_id) AS unique_users,
  AVG(time_on_site) AS avg_time_on_site,
  SUM(CASE WHEN had_download_event THEN 1 ELSE 0 END) / COUNT(*) AS conversion_rate
FROM `moz-fx-data-shared-prod.firefoxdotcom_derived.ga_sessions_v2`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 14 DAY)
  AND first_experiment_id_from_event_params IS NOT NULL
GROUP BY experiment_id, branch
ORDER BY experiment_id, branch;
```

## Key Considerations

- Sessions without a non-null `ga_session_id` are excluded from this table
- Data is updated incrementally with a 3-day lookback window to capture late-arriving events
- When a `session_start` event is missing, the table falls back to the first event timestamp and country
- The table contains both single-value fields (first reported) and arrays (all distinct values) for flexibility in analysis