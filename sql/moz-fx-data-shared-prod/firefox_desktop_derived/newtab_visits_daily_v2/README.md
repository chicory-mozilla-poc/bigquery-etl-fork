# Firefox Desktop Newtab Visits Daily (v2)

## Table Overview

**Table Name:** `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`

**Description:** Daily aggregation of Firefox Desktop newtab visit metrics, capturing user interactions with content, topsites, search, and widgets. Each row represents a unique newtab visit session identified by `newtab_visit_id` for a given day.

**Owner:** gkatre@mozilla.com

**Scheduling:** Daily incremental ETL via bqetl_newtab DAG

**Partitioning:** Day-level partitioning on `submission_date` (partition filter required)

**Clustering:** Optimized for queries filtering by `channel`, `country`, and `newtab_category`

**Data Retention:** 775 days

## Data Source

This table is derived from the `firefox_desktop_stable.newtab_v1` ping, which captures user interactions with the Firefox newtab page. Events are unnested and aggregated by visit session (`newtab_visit_id`) to create a comprehensive view of user behavior during each newtab visit.

## Key Metrics Categories

### 1. User Configuration
- **Content Settings:** `organic_content_enabled`, `sponsored_content_enabled`
- **Topsite Settings:** `organic_topsites_enabled`, `sponsored_topsites_enabled`
- **Search Settings:** `newtab_search_enabled`, `default_search_engine`
- **Widget Settings:** `newtab_weather_enabled`, `topsite_rows`
- **UI Settings:** `homepage_category`, `newtab_category`, `is_default_ui`

### 2. Content Engagement (Pocket Stories)
- **Clicks:** `any_content_click_count`, `organic_content_click_count`, `sponsored_content_click_count`
- **Impressions:** `any_content_impression_count`, `organic_content_impression_count`, `sponsored_content_impression_count`
- **Interactions:** `any_content_interaction_count`, `organic_content_interaction_count`, `sponsored_content_interaction_count`
- **Feedback:** `content_thumbs_up_count`, `content_thumbs_down_count`
- **Dismissals:** `organic_content_dismissal_count`, `sponsored_content_dismissal_count`

### 3. Topsite Engagement
- **Clicks:** `any_topsite_click_count`, `organic_topsite_click_count`, `sponsored_topsite_click_count`
- **Impressions:** `any_topsite_impression_count`, `organic_topsite_impression_count`, `sponsored_topsite_impression_count`
- **Interactions:** `any_topsite_interaction_count`, `organic_topsite_interaction_count`, `sponsored_topsite_interaction_count`
- **Dismissals:** `organic_topsite_dismissal_count`, `sponsored_topsite_dismissal_count`

### 4. Search & Advertising
- **Search Activity:** `is_search_issued`, `search_interaction_count`
- **Ad Performance:** `search_ad_click_count`, `search_ad_impression_count`, `is_search_ad_click`

### 5. Widget & Other Features
- **Widget Metrics:** `widget_interaction_count`, `widget_impression_count`, `is_widget_interaction`
- **Wallpaper:** `is_wallpaper_interaction`
- **Other Features:** `other_interaction_count`, `other_impression_count`, `is_section`

### 6. Visit Metadata
- **Session Info:** `newtab_visit_id`, `is_newtab_opened`, `newtab_visit_duration`
- **Window Dimensions:** `newtab_window_inner_height`, `newtab_window_inner_width`
- **Geography:** `country`, `geo_subdivision`
- **Experiments:** `experiments` (nested RECORD with branch and enrollment data)

## ETL Logic Summary

The query performs the following transformations:

1. **Event Unnesting:** Extracts and flattens events from `newtab_v1` ping for the target submission date
2. **Event Filtering:** Selects relevant event categories: `newtab.search`, `newtab.search.ad`, `pocket`, `topsites`, and `newtab` widget/wallpaper events
3. **Default UI Detection:** Uses `mozfun.newtab.is_default_ui_v1()` to identify visits with default Firefox UI
4. **Visit Aggregation:** Groups events by `submission_date`, `client_id`, and `newtab_visit_id`
5. **Metric Computation:**
   - Boolean flags: `LOGICAL_OR()` to detect if an event occurred during the visit
   - Count metrics: `COUNTIF()` to sum occurrences of specific events
   - Duration: Calculated as difference between close and open event timestamps
   - Configuration: Uses `ANY_VALUE()` for user settings that don't vary within a visit

## Downstream Analysis Suggestions

### 1. Sponsored Content ROI Analysis
```sql
SELECT
  submission_date,
  SUM(sponsored_content_click_count) AS total_sponsored_clicks,
  SUM(sponsored_content_impression_count) AS total_sponsored_impressions,
  SAFE_DIVIDE(SUM(sponsored_content_click_count), SUM(sponsored_content_impression_count)) AS ctr,
  COUNT(DISTINCT CASE WHEN sponsored_content_click_count > 0 THEN client_id END) AS unique_clickers
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND is_default_ui = TRUE
GROUP BY submission_date
ORDER BY submission_date DESC;
```
**Use Case:** Track click-through rates and engagement for sponsored Pocket content to measure advertising effectiveness.

### 2. User Engagement Funnel by Geography
```sql
SELECT
  country,
  COUNT(DISTINCT client_id) AS total_users,
  COUNTIF(is_newtab_opened) AS newtab_opens,
  COUNTIF(is_search_issued) AS search_issued,
  COUNTIF(is_content_click) AS content_clicks,
  COUNTIF(is_topsite_click) AS topsite_clicks,
  AVG(newtab_visit_duration / 1000) AS avg_visit_seconds
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`
WHERE submission_date = CURRENT_DATE() - 1
  AND is_default_ui = TRUE
GROUP BY country
HAVING total_users >= 1000
ORDER BY total_users DESC
LIMIT 20;
```
**Use Case:** Identify geographic differences in newtab engagement patterns and optimize content delivery strategies.

### 3. Feature Adoption & Interaction Patterns
```sql
SELECT
  CASE
    WHEN newtab_weather_enabled THEN 'Weather Enabled'
    ELSE 'Weather Disabled'
  END AS weather_status,
  COUNT(DISTINCT client_id) AS users,
  AVG(widget_interaction_count) AS avg_widget_interactions,
  AVG(any_interaction_count) AS avg_total_interactions,
  AVG(newtab_visit_duration / 1000) AS avg_visit_seconds
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
  AND is_default_ui = TRUE
GROUP BY weather_status;
```
**Use Case:** Compare engagement metrics between users who enable specific features (e.g., weather widget) versus those who don't.

### 4. Experiment Impact Analysis
```sql
SELECT
  exp.key AS experiment_name,
  exp.value.branch AS experiment_branch,
  COUNT(DISTINCT client_id) AS users,
  AVG(sponsored_content_click_count) AS avg_sponsored_clicks,
  AVG(organic_content_click_count) AS avg_organic_clicks,
  SAFE_DIVIDE(SUM(sponsored_content_click_count), SUM(sponsored_content_impression_count)) AS sponsored_ctr
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`,
  UNNEST(experiments) AS exp
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 14 DAY)
  AND is_default_ui = TRUE
  AND exp.key LIKE '%newtab%'
GROUP BY experiment_name, experiment_branch
ORDER BY users DESC;
```
**Use Case:** Measure the impact of A/B experiments on user engagement with newtab content and features.

## Important Notes

- **Default UI Filter:** Most interaction metrics are only computed when `is_default_ui = TRUE`, ensuring accurate measurement of standard newtab behavior
- **Visit-Level Granularity:** Each row represents a single newtab visit, enabling session-level analysis
- **Sponsored vs. Organic:** The table distinguishes between sponsored and organic content/topsites throughout, enabling revenue and content strategy analysis
- **Partition Requirement:** Queries must include a `submission_date` filter due to partition requirements
- **Nested Experiments:** The `experiments` field is a RECORD type; use `UNNEST()` to access experiment details

## Related Tables

- **Source:** `firefox_desktop_stable.newtab_v1` - Raw newtab ping events
- **Client-Level:** `telemetry_derived.clients_daily_v6` - For joining with broader client activity metrics
- **Search:** `search_derived.search_clients_daily_v8` - For correlating with search behavior