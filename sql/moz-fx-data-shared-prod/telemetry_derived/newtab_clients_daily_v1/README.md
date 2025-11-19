# Newtab Clients Daily

## Overview
Daily aggregation of Firefox desktop new tab page interactions at the client level. This table consolidates user engagement with various new tab features including top sites, Pocket recommendations, wallpapers, weather widget, and search interactions. Derived from `newtab_visits_v1` with daily partitioning.

## Key Features
- **Granularity**: One row per client per day
- **Partitioning**: Daily partition on `submission_date` with 775-day retention
- **Clustering**: Optimized by `channel` and `country_code`
- **Update Schedule**: Daily via `bqetl_newtab_late_morning` DAG

## Schema Highlights
- **Identity**: `client_id`, `legacy_telemetry_client_id`, `profile_group_id`
- **Configuration**: Browser settings, feature flags, experiment enrollment
- **Engagement Metrics**: Clicks, impressions, saves, dismissals across content types
- **Search Metrics**: Search counts and ad interaction metrics
- **Content Surfaces**: Top sites, Pocket stories, wallpapers, weather widget
- **User Segmentation**: Activity segment, new profile indicator, topic preferences

## Data Sources
- **Primary**: `moz-fx-data-shared-prod.telemetry_derived.newtab_visits_v1`
- **Aggregation**: Daily rollup with UNNEST operations on interaction arrays

## Common Use Cases

### 1. Feature Adoption Analysis
Analyze adoption rates of new tab features across user segments:
```sql
SELECT 
  activity_segment,
  COUNTIF(pocket_enabled) / COUNT(*) AS pocket_enabled_rate,
  COUNTIF(topsites_sponsored_enabled) / COUNT(*) AS sponsored_tiles_enabled_rate,
  COUNTIF(newtab_weather_widget_enabled) / COUNT(*) AS weather_widget_enabled_rate
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY activity_segment
```

### 2. Sponsored Content Performance
Compare engagement with sponsored vs organic content:
```sql
SELECT 
  submission_date,
  SUM(sponsored_pocket_clicks) / NULLIF(SUM(sponsored_pocket_impressions), 0) AS sponsored_ctr,
  SUM(organic_pocket_clicks) / NULLIF(SUM(organic_pocket_impressions), 0) AS organic_ctr,
  SUM(sponsored_topsite_tile_clicks) / NULLIF(SUM(sponsored_topsite_tile_impressions), 0) AS sponsored_tile_ctr
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1`
WHERE submission_date BETWEEN '2024-01-01' AND '2024-01-31'
  AND channel = 'release'
GROUP BY submission_date
```

### 3. User Engagement Segmentation
Identify highly engaged users based on non-search interactions:
```sql
SELECT 
  CASE 
    WHEN non_search_engagement_count >= 10 THEN 'High Engagement'
    WHEN non_search_engagement_count >= 5 THEN 'Medium Engagement'
    WHEN non_search_engagement_count > 0 THEN 'Low Engagement'
    ELSE 'No Engagement'
  END AS engagement_tier,
  COUNT(DISTINCT client_id) AS client_count,
  AVG(newtab_visit_count) AS avg_visits
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1`
WHERE submission_date = CURRENT_DATE() - 1
GROUP BY engagement_tier
```

### 4. Search Revenue Attribution
Analyze search ad performance by geography and channel:
```sql
SELECT 
  country_code,
  channel,
  SUM(tagged_search_ad_clicks) AS total_ad_clicks,
  SUM(tagged_search_ad_impressions) AS total_ad_impressions,
  SUM(tagged_search_ad_clicks) / NULLIF(SUM(tagged_search_ad_impressions), 0) AS ad_ctr
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
  AND country_code != '??'
GROUP BY country_code, channel
ORDER BY total_ad_clicks DESC
```

## Important Notes
- Requires partition filter on `submission_date` for query execution
- `topsites_sponsored_tiles_configured` only available for Firefox 123+ (2024-02-20 onwards)
- List card metrics introduced in recent Firefox versions
- COALESCE applied to interaction metrics (defaults to 0) when no interactions occurred

## Owners
- cbeck@mozilla.com
- mbowerman@mozilla.com

## Labels
- **Application**: firefox
- **Type**: client_level
- **Update Frequency**: daily
- **Incremental**: true

---
*Last Updated: Auto-generated table documentation*