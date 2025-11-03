# Newtab Visits Daily (newtab_visits_daily_v2)

## Overview

This table provides daily-level metrics for Firefox Desktop newtab page visits, with one row per visit (identified by `newtab_visit_id`) per day. It captures comprehensive user interaction data including content clicks, search activity, sponsored tile engagement, widget usage, and personalization settings. The data is derived from the `firefox_desktop_stable.newtab_v1` ping and aggregates events within each visit session.

## Data Characteristics

**Granularity:** One row per newtab visit per day
**Update Frequency:** Daily
**Partitioning:** Partitioned by `submission_date` (day-level)
**Clustering:** Optimized by `channel`, `country`, and `newtab_category`
**Retention:** 775 days
**Data Source:** `firefox_desktop_stable.newtab_v1` ping

## Key Features

### User Engagement Metrics
- **Content Interactions:** Tracks Pocket story clicks, impressions, and dismissals (both organic and sponsored)
- **Topsite Activity:** Monitors topsite tile clicks, impressions, and user dismissals
- **Search Behavior:** Captures search queries issued and search ad interactions
- **Widget Usage:** Records weather, list, and timer widget interactions and impressions
- **Personalization:** Tracks wallpaper changes, topic selections, and section management

### Configuration & Settings
- User preferences for sponsored vs. organic content
- Homepage and newtab appearance categories
- Search engine configurations (default and private)
- Topsite tile configuration (rows and sponsored tiles)
- Blocked sponsor tracking

### Experimental Data
- Active experiment enrollments with branch assignments
- Experiment metadata including type and enrollment IDs

## Visit Session Logic

The table groups events by `newtab_visit_id`, which uniquely identifies a single newtab session from open to close. The `is_default_ui` flag filters events to only count those occurring in the default UI configuration, ensuring consistent measurement across different newtab customizations.

**Visit Duration Calculation:**
```sql
MAX(close_event_timestamp) - MIN(open_event_timestamp)
```

## Downstream Analysis Suggestions

### 1. Sponsored Content Performance Analysis
Analyze the effectiveness of sponsored content and topsites by comparing click-through rates, impression counts, and dismissal patterns:
```sql
SELECT 
  submission_date,
  country,
  channel,
  SUM(sponsored_content_impression_count) as total_sponsored_impressions,
  SUM(sponsored_content_click_count) as total_sponsored_clicks,
  SAFE_DIVIDE(SUM(sponsored_content_click_count), SUM(sponsored_content_impression_count)) as ctr,
  SUM(sponsored_content_dismissal_count) as total_dismissals
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND is_default_ui
GROUP BY submission_date, country, channel
ORDER BY submission_date DESC;
```

### 2. User Engagement Funnel Analysis
Build engagement funnels to understand user journey from newtab open to content interaction:
```sql
WITH user_engagement AS (
  SELECT
    client_id,
    COUNTIF(is_newtab_opened) as newtab_opens,
    COUNTIF(is_content_impression) as saw_content,
    COUNTIF(is_content_interaction) as interacted_with_content,
    COUNTIF(is_search_issued) as performed_searches
  FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`
  WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
  GROUP BY client_id
)
SELECT
  COUNT(*) as total_users,
  COUNTIF(newtab_opens > 0) as users_opened_newtab,
  COUNTIF(saw_content > 0) as users_saw_content,
  COUNTIF(interacted_with_content > 0) as users_interacted,
  COUNTIF(performed_searches > 0) as users_searched
FROM user_engagement;
```

### 3. Feature Adoption and Configuration Analysis
Examine adoption rates of different newtab features and user configuration preferences:
```sql
SELECT
  newtab_category,
  COUNT(DISTINCT client_id) as unique_users,
  AVG(CAST(organic_content_enabled AS INT64)) as pct_organic_enabled,
  AVG(CAST(sponsored_content_enabled AS INT64)) as pct_sponsored_enabled,
  AVG(CAST(newtab_weather_enabled AS INT64)) as pct_weather_enabled,
  AVG(topsite_rows) as avg_topsite_rows,
  SUM(widget_interaction_count) as total_widget_interactions
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY newtab_category
ORDER BY unique_users DESC;
```

### 4. Experiment Impact Assessment
Evaluate the impact of active experiments on key engagement metrics:
```sql
WITH experiment_users AS (
  SELECT
    client_id,
    exp.key as experiment_key,
    exp.value.branch as experiment_branch,
    AVG(any_content_click_count) as avg_content_clicks,
    AVG(any_topsite_click_count) as avg_topsite_clicks,
    AVG(newtab_visit_duration) / 1000 as avg_duration_seconds,
    COUNT(*) as visit_count
  FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`,
    UNNEST(experiments) as exp
  WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 14 DAY)
    AND is_default_ui
  GROUP BY client_id, experiment_key, experiment_branch
)
SELECT
  experiment_key,
  experiment_branch,
  COUNT(DISTINCT client_id) as enrolled_users,
  AVG(avg_content_clicks) as avg_content_clicks_per_visit,
  AVG(avg_topsite_clicks) as avg_topsite_clicks_per_visit,
  AVG(avg_duration_seconds) as avg_visit_duration_sec
FROM experiment_users
GROUP BY experiment_key, experiment_branch
ORDER BY enrolled_users DESC;
```

## Important Notes

- **Default UI Filtering:** Most boolean flags and count metrics only include visits with `is_default_ui = TRUE` to ensure consistent measurement
- **Visit Aggregation:** Events are aggregated by `newtab_visit_id`, so multiple events within a single visit are rolled up
- **Null Values:** Configuration fields may be NULL if not set by the user or not applicable to their browser version
- **Array Fields:** `experiments` and `newtab_blocked_sponsors` are repeated/array fields requiring UNNEST for analysis

## Owner

**Primary Contact:** gkatre@mozilla.com
**Team:** Newtab
**Schedule:** Daily via `bqetl_newtab` DAG