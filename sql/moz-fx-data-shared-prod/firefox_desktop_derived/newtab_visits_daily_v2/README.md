# Firefox Desktop Newtab Visits Daily

## Overview

This table provides daily metrics for Firefox Desktop newtab (new tab page) visits. Each row represents a unique newtab visit session, capturing user interactions with various newtab features including sponsored and organic content, topsites, search functionality, widgets, and customization options.

## Data Source

The table is derived from the `firefox_desktop_stable.newtab_v1` ping, processing events related to newtab opens, closes, searches, content impressions, clicks, and user interactions. The ETL aggregates event-level telemetry into visit-level metrics.

## Key Metrics

- **Visit Identification**: Each visit is uniquely identified by `newtab_visit_id`
- **Content Engagement**: Tracks both organic and sponsored content from Pocket (clicks, impressions, dismissals)
- **Topsite Activity**: Monitors organic and sponsored topsite interactions
- **Search Behavior**: Captures search issuance and ad engagement
- **Widgets**: Tracks weather, timer, and list widget usage
- **Customization**: Records wallpaper, topic selection, and section preferences

## Table Characteristics

- **Granularity**: One row per newtab visit per day
- **Partitioning**: Daily partitioning on `submission_date` (required filter)
- **Clustering**: Optimized by `channel`, `country`, `newtab_category`
- **Update Schedule**: Daily via `bqetl_newtab` DAG
- **Data Retention**: 775 days

## Default UI Context

Many metrics are conditioned on `is_default_ui = TRUE`, meaning they only capture interactions when the newtab page was opened in the default Firefox UI configuration (not in custom homepage settings).

## Downstream Analysis Suggestions

### 1. Sponsored Content Performance Analysis
Analyze the effectiveness of sponsored content and topsites:
```sql
SELECT 
  submission_date,
  channel,
  country,
  COUNT(DISTINCT client_id) as users,
  SUM(sponsored_content_impression_count) as sponsored_impressions,
  SUM(sponsored_content_click_count) as sponsored_clicks,
  SAFE_DIVIDE(SUM(sponsored_content_click_count), SUM(sponsored_content_impression_count)) as ctr
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND is_default_ui = TRUE
  AND sponsored_content_enabled = TRUE
GROUP BY 1,2,3
ORDER BY submission_date DESC;
```

### 2. User Engagement Funnel
Build a funnel showing newtab visit to content interaction:
```sql
SELECT
  submission_date,
  COUNT(DISTINCT newtab_visit_id) as total_visits,
  COUNT(DISTINCT CASE WHEN is_content_impression THEN newtab_visit_id END) as visits_with_impressions,
  COUNT(DISTINCT CASE WHEN is_content_click THEN newtab_visit_id END) as visits_with_clicks,
  COUNT(DISTINCT CASE WHEN is_search_issued THEN newtab_visit_id END) as visits_with_search
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
  AND is_default_ui = TRUE
GROUP BY 1
ORDER BY 1 DESC;
```

### 3. Feature Adoption and Retention
Track adoption of newtab features over time:
```sql
SELECT
  submission_date,
  COUNTIF(newtab_weather_enabled) as weather_enabled_users,
  COUNTIF(sponsored_content_enabled) as sponsored_stories_enabled,
  COUNTIF(sponsored_topsites_enabled) as sponsored_topsites_enabled,
  COUNTIF(is_widget_interaction) as widget_interactions,
  COUNT(DISTINCT client_id) as total_users
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY 1
ORDER BY 1 DESC;
```

### 4. Experiment Impact Analysis
Compare metrics across experiment branches:
```sql
WITH experiment_users AS (
  SELECT
    submission_date,
    client_id,
    exp.value.branch as experiment_branch,
    AVG(any_content_click_count) as avg_content_clicks,
    AVG(search_interaction_count) as avg_searches,
    AVG(newtab_visit_duration) as avg_duration_ms
  FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`,
  UNNEST(experiments) as exp
  WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 14 DAY)
    AND exp.key = 'your-experiment-id'
  GROUP BY 1,2,3
)
SELECT
  experiment_branch,
  AVG(avg_content_clicks) as mean_content_clicks,
  AVG(avg_searches) as mean_searches,
  AVG(avg_duration_ms) / 1000 as mean_duration_seconds
FROM experiment_users
GROUP BY 1;
```

## Important Notes

- Metrics with `_count` suffix represent aggregated counts within a single visit
- Boolean `is_*` fields indicate whether an event occurred at least once during the visit
- All interaction metrics are filtered to `is_default_ui = TRUE` for consistency
- Visit duration may be NULL if close event was not captured
- The `experiments` field is a repeated RECORD containing all active experiments for the visit

## Contact

- **Owner**: gkatre@mozilla.com
- **DAG**: bqetl_newtab
- **Schedule**: Daily incremental load