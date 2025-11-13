# New Tab Clients Daily

## Overview
A daily aggregation of Firefox desktop new tab page interactions and configurations for each client. This table consolidates visit-level data from `newtab_visits_v1` into client-level daily summaries, partitioned by submission date.

## Data Source
Aggregated from `moz-fx-data-shared-prod.telemetry_derived.newtab_visits_v1`

## Key Features
- **Granularity**: One row per client per day
- **Partitioning**: Daily partition by `submission_date`
- **Retention**: 775 days
- **Clustering**: Optimized by `channel` and `country_code`

## Data Categories

### Client & Configuration
- Client identifiers and profile information
- Browser version, OS, locale, and geographic data
- Feature enablement flags (Pocket, Top Sites, Weather Widget)
- Experiment enrollments and branch assignments

### Search Activity
- Search counts and ad interactions
- Tagged vs. follow-on search metrics
- Ad impressions and click-through data

### Content Engagement
- **Top Sites**: Clicks, impressions, and dismissals (sponsored vs. organic)
- **Pocket**: Story interactions including clicks, saves, impressions, and feedback
- **List Cards**: New Pocket list format engagement metrics
- **Wallpaper**: Customization interactions and feature discovery
- **Weather Widget**: Usage patterns and display preferences
- **Topics**: Personalization preferences and selection behavior

### Engagement Metrics
- Visit counts with different UI configurations
- Non-search and non-impression engagement summaries
- Overall dismissal and interaction counts

## Suggested Downstream Analysis

### 1. Sponsored Content Performance Analysis
Analyze the effectiveness of sponsored content across different surfaces:
```sql
SELECT 
  submission_date,
  channel,
  SAFE_DIVIDE(SUM(sponsored_topsite_tile_clicks), SUM(sponsored_topsite_tile_impressions)) AS sponsored_tile_ctr,
  SAFE_DIVIDE(SUM(sponsored_pocket_clicks), SUM(sponsored_pocket_impressions)) AS sponsored_pocket_ctr,
  SAFE_DIVIDE(SUM(sponsored_list_card_clicks), SUM(sponsored_list_card_impressions)) AS sponsored_list_ctr
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY 1, 2
ORDER BY 1 DESC
```

### 2. Feature Adoption and Engagement Cohort Analysis
Track how new profile adoption differs from existing users:
```sql
SELECT
  is_new_profile,
  activity_segment,
  COUNT(DISTINCT client_id) AS clients,
  AVG(newtab_visit_count) AS avg_visits,
  AVG(non_search_engagement_count) AS avg_engagement,
  COUNTIF(topic_preferences_set) / COUNT(*) AS topic_adoption_rate,
  COUNTIF(pocket_is_signed_in) / COUNT(*) AS pocket_signin_rate
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
GROUP BY 1, 2
```

### 3. Weather Widget Usage Patterns
Understand weather widget engagement and error rates:
```sql
SELECT
  submission_date,
  country_code,
  COUNT(DISTINCT client_id) AS clients_with_weather_enabled,
  SUM(weather_widget_impressions) AS total_impressions,
  SUM(weather_widget_clicks) AS total_clicks,
  SAFE_DIVIDE(SUM(weather_widget_load_errors), SUM(weather_widget_impressions)) AS error_rate,
  SUM(weather_widget_change_display_to_detailed) AS detailed_view_switches,
  SUM(weather_widget_location_selected) AS location_changes
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 14 DAY)
  AND newtab_weather_widget_enabled = TRUE
GROUP BY 1, 2
HAVING clients_with_weather_enabled > 100
```

### 4. UI Configuration Impact on Engagement
Compare engagement between default and customized UI configurations:
```sql
SELECT
  submission_date,
  CASE 
    WHEN visits_with_default_ui > visits_with_non_default_ui THEN 'Default UI Dominant'
    WHEN visits_with_non_default_ui > visits_with_default_ui THEN 'Custom UI Dominant'
    ELSE 'Mixed'
  END AS ui_preference,
  COUNT(DISTINCT client_id) AS clients,
  AVG(SAFE_DIVIDE(non_search_engagement_count, newtab_visit_count)) AS engagement_per_visit,
  AVG(searches) AS avg_searches,
  AVG(newtab_dismissal_count) AS avg_dismissals
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND newtab_visit_count > 0
GROUP BY 1, 2
ORDER BY 1 DESC, 2
```

## Important Notes
- Partition filter on `submission_date` is required for queries
- Metrics are aggregated using COALESCE to handle null values (default to 0)
- `topsites_sponsored_tiles_configured` only available for Firefox 123+ (released 2024-02-20)
- Use clustering columns in WHERE clauses for optimal query performance

## Owners
- cbeck@mozilla.com
- mbowerman@mozilla.com