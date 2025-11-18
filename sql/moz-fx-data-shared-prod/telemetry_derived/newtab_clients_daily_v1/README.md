# Newtab Clients Daily

## Overview

The `newtab_clients_daily_v1` table provides a comprehensive daily aggregation of Firefox new tab page activity for each client. This table consolidates user interactions across multiple new tab surfaces including search, top sites tiles, Pocket recommendations, weather widgets, wallpaper customization, and topic preferences.

**Source:** Derived from `telemetry_derived.newtab_visits_v1`

**Grain:** One row per client per day

**Partitioning:** Daily partitioned by `submission_date` with 775-day retention

**Clustering:** Optimized by `channel` and `country_code` for efficient querying

## Key Features

### Tracked Surfaces & Interactions

- **Search Activity:** Tracks searches, ad impressions, and ad clicks (both initial and follow-on)
- **Top Sites:** Monitors tile impressions, clicks, and dismissals (separated by sponsored vs organic)
- **Pocket Integration:** Captures story impressions, clicks, saves, dismissals, and ratings
- **Weather Widget:** Records impressions, clicks, display mode changes, and errors
- **Wallpaper Customization:** Tracks wallpaper selection and feature engagement
- **Topic Preferences:** Monitors topic selection interface interactions and updates
- **List Cards:** New Pocket list-style card format with separate metrics

### User Configuration

- Browser and OS details with versioning
- Feature enable/disable flags (Pocket, top sites, weather, sponsorship)
- Search engine preferences (default and private)
- Experiment enrollment tracking with branch assignments
- UI configuration and customization states

### Engagement Metrics

- Visit counts with engagement classification
- Separation of search vs non-search engagement
- Default UI vs customized UI tracking
- Aggregate engagement and dismissal counters

## ETL Logic

The query aggregates visit-level data through several CTEs:

1. **visits_data_base:** Filters newtab_visits for the target date
2. **visits_data:** Aggregates visit counts and configuration at client level
3. **search_data:** Sums search interactions using UNNEST
4. **tiles_data:** Aggregates top site tile interactions
5. **pocket_data:** Sums Pocket story and list card interactions
6. **wallpaper_data:** Aggregates wallpaper customization events
7. **weather_data:** Sums weather widget interactions
8. **topic_selection_data:** Aggregates topic preference activities
9. **joined:** LEFT JOINs all CTEs with COALESCE for null handling
10. **Final SELECT:** Calculates derived engagement metrics

## Downstream Analysis Suggestions

### 1. Sponsored Content Performance Analysis

```sql
SELECT
  submission_date,
  COUNT(DISTINCT client_id) AS active_users,
  AVG(sponsored_topsite_tile_impressions) AS avg_sponsored_tile_impressions,
  AVG(sponsored_pocket_impressions) AS avg_sponsored_pocket_impressions,
  SUM(sponsored_topsite_tile_clicks) / NULLIF(SUM(sponsored_topsite_tile_impressions), 0) AS sponsored_tile_ctr,
  SUM(sponsored_pocket_clicks) / NULLIF(SUM(sponsored_pocket_impressions), 0) AS sponsored_pocket_ctr
FROM moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND topsites_sponsored_enabled = TRUE
GROUP BY submission_date
ORDER BY submission_date DESC;
```

Analyze click-through rates, impression volumes, and user engagement with sponsored content across both top sites and Pocket surfaces.

### 2. Feature Adoption & Configuration Trends

```sql
SELECT
  submission_date,
  channel,
  COUNTIF(pocket_enabled) / COUNT(*) AS pocket_enabled_rate,
  COUNTIF(topsites_sponsored_enabled) / COUNT(*) AS sponsored_tiles_enabled_rate,
  COUNTIF(newtab_weather_widget_enabled) / COUNT(*) AS weather_enabled_rate,
  COUNTIF(topic_preferences_set) / COUNT(*) AS topic_prefs_set_rate,
  AVG(topsites_rows) AS avg_topsite_rows
FROM moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY submission_date, channel
ORDER BY submission_date DESC, channel;
```

Track how users configure their new tab experience across different release channels and identify feature adoption patterns over time.

### 3. User Engagement Cohort Analysis

```sql
SELECT
  activity_segment,
  channel,
  COUNT(DISTINCT client_id) AS users,
  AVG(newtab_visit_count) AS avg_visits,
  AVG(non_search_engagement_count) AS avg_engagements,
  AVG(searches) AS avg_searches,
  AVG(newtab_dismissal_count) AS avg_dismissals,
  SUM(pocket_saves) AS total_pocket_saves,
  SUM(topsite_tile_clicks) AS total_tile_clicks
FROM moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1
WHERE submission_date = CURRENT_DATE() - 1
  AND activity_segment IS NOT NULL
GROUP BY activity_segment, channel
ORDER BY activity_segment, channel;
```

Segment users by activity level to understand engagement patterns, feature usage, and content interaction differences across user cohorts.

### 4. Experiment Impact Assessment

```sql
WITH experiment_clients AS (
  SELECT
    client_id,
    submission_date,
    exp.key AS experiment_id,
    exp.value.branch AS branch,
    newtab_visit_count,
    non_search_engagement_count,
    sponsored_topsite_tile_clicks,
    sponsored_pocket_clicks
  FROM moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1
  CROSS JOIN UNNEST(experiments) AS exp
  WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
    AND exp.key = 'your-experiment-slug'
)
SELECT
  branch,
  COUNT(DISTINCT client_id) AS enrolled_users,
  AVG(newtab_visit_count) AS avg_visits,
  AVG(non_search_engagement_count / NULLIF(newtab_visit_count, 0)) AS engagement_rate,
  AVG(sponsored_topsite_tile_clicks) AS avg_tile_clicks,
  AVG(sponsored_pocket_clicks) AS avg_pocket_clicks
FROM experiment_clients
GROUP BY branch
ORDER BY branch;
```

Evaluate experiment performance by comparing key metrics across treatment branches, enabling data-driven decisions for new tab features.

## Notes

- **Scheduling:** Runs daily via the `bqetl_newtab_late_morning` DAG
- **New Telemetry:** `topsites_sponsored_tiles_configured` available only for Firefox 123+ (2024/02/20 onwards)
- **List Cards:** New Pocket card format with separate metrics introduced recently
- **Partition Filtering:** Required due to large table size; always include `submission_date` filter
- **NULL Handling:** Interaction counts are coalesced to 0 for clients with no activity on specific surfaces