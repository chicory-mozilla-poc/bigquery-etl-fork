# Newtab Clients Daily

## Overview

The `newtab_clients_daily_v1` table provides a comprehensive daily aggregation of Firefox desktop new tab page interactions and configurations for each client. This table aggregates data from `newtab_visits_v1`, consolidating all new tab page activities, user preferences, and engagement metrics into a single client-level daily record.

## Table Details

- **Project**: moz-fx-data-shared-prod
- **Dataset**: telemetry_derived
- **Table**: newtab_clients_daily_v1
- **Partitioning**: Daily partition on `submission_date` (775 days retention)
- **Clustering**: `channel`, `country_code`
- **Schedule**: Daily (bqetl_newtab_late_morning DAG)

## Key Features

This table aggregates multiple dimensions of new tab page usage:

1. **Client Demographics**: Browser version, OS, locale, country, channel
2. **Feature Configuration**: Settings for Pocket, top sites, weather widget, sponsored content
3. **Search Behavior**: Search counts, ad clicks and impressions (both initial and follow-on)
4. **Top Sites Engagement**: Clicks, impressions, and dismissals for organic and sponsored tiles
5. **Pocket Interactions**: Clicks, saves, impressions, and feedback on recommendations
6. **Weather Widget Usage**: Impressions, clicks, display preferences, and errors
7. **Wallpaper Customization**: Selection, category browsing, and feature engagement
8. **Topic Personalization**: Topic preference settings and updates
9. **Experiment Tracking**: Active experiments and branch assignments

## Data Aggregation Logic

The ETL query performs the following aggregations:

- **Client-level aggregation**: All visit-level data is grouped by `client_id` and `submission_date`
- **Configuration fields**: Uses `ANY_VALUE()` to capture client settings (assumes consistency within a day)
- **Boolean flags**: Uses `LOGICAL_OR()` to indicate if a behavior occurred at least once
- **Interaction metrics**: Uses `SUM()` across all visits to count total engagements
- **Array unnesting**: Expands nested interaction arrays from visits for accurate metric calculation
- **Default value handling**: Uses `COALESCE()` to set 0 for NULL interaction counts

## Downstream Analysis Suggestions

### 1. Sponsored Content Performance Analysis

Analyze the effectiveness of sponsored content across different surfaces:

```sql
SELECT
  submission_date,
  SAFE_DIVIDE(SUM(sponsored_topsite_tile_clicks), SUM(sponsored_topsite_tile_impressions)) AS sponsored_tiles_ctr,
  SAFE_DIVIDE(SUM(sponsored_pocket_clicks), SUM(sponsored_pocket_impressions)) AS sponsored_pocket_ctr,
  SAFE_DIVIDE(SUM(organic_topsite_tile_clicks), SUM(organic_topsite_tile_impressions)) AS organic_tiles_ctr,
  SAFE_DIVIDE(SUM(organic_pocket_clicks), SUM(organic_pocket_impressions)) AS organic_pocket_ctr
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY submission_date
ORDER BY submission_date;
```

### 2. Feature Adoption and Configuration Trends

Track which new tab features are most commonly enabled and their usage patterns:

```sql
SELECT
  submission_date,
  COUNTIF(pocket_enabled) AS pocket_enabled_clients,
  COUNTIF(topsites_sponsored_enabled) AS sponsored_tiles_enabled,
  COUNTIF(newtab_weather_widget_enabled) AS weather_enabled,
  COUNTIF(topic_preferences_set) AS topic_prefs_set,
  AVG(CASE WHEN newtab_weather_widget_enabled THEN weather_widget_impressions END) AS avg_weather_impressions
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY submission_date
ORDER BY submission_date;
```

### 3. Search Revenue Attribution

Analyze search ad performance and revenue attribution from new tab interactions:

```sql
SELECT
  country_code,
  default_search_engine,
  SUM(searches) AS total_searches,
  SUM(tagged_search_ad_clicks) AS tagged_clicks,
  SUM(tagged_search_ad_impressions) AS tagged_impressions,
  SUM(tagged_follow_on_search_ad_clicks) AS follow_on_clicks,
  SAFE_DIVIDE(SUM(tagged_search_ad_clicks), SUM(searches)) AS search_to_ad_click_rate
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1`
WHERE submission_date = CURRENT_DATE() - 1
  AND searches > 0
GROUP BY country_code, default_search_engine
ORDER BY total_searches DESC
LIMIT 20;
```

### 4. User Engagement Segmentation

Segment users by engagement level and analyze their interaction patterns:

```sql
SELECT
  activity_segment,
  COUNT(DISTINCT client_id) AS client_count,
  AVG(newtab_visit_count) AS avg_visits,
  AVG(non_search_engagement_count) AS avg_non_search_engagement,
  AVG(searches) AS avg_searches,
  COUNTIF(wallpaper_clicks > 0) / COUNT(*) AS pct_customizing_wallpaper,
  COUNTIF(topic_preferences_set) / COUNT(*) AS pct_with_topic_prefs
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
GROUP BY activity_segment
ORDER BY client_count DESC;
```

## Important Notes

- The `topsites_sponsored_tiles_configured` field is only available for Firefox 123+ (released 2024/02/20)
- Experiment data is stored as a RECORD array with nested fields for branch and enrollment details
- All interaction metrics are aggregated across multiple visits per client per day
- NULL values in interaction counts are coalesced to 0 for easier analysis
- Partitioning by `submission_date` is required for query efficiency

## Related Tables

- `telemetry_derived.newtab_visits_v1`: Source table with visit-level granularity
- `telemetry_derived.clients_daily_v6`: General Firefox telemetry client-level data

## Owners

- cbeck@mozilla.com
- mbowerman@mozilla.com