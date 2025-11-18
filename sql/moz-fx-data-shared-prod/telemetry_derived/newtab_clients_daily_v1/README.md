# Newtab Clients Daily (newtab_clients_daily_v1)

## Overview

The `newtab_clients_daily_v1` table provides a comprehensive daily aggregation of Firefox New Tab page interactions for each desktop client. This derived table consolidates visit-level data from `newtab_visits_v1` into client-level daily summaries, enabling analysis of user engagement patterns, feature adoption, and monetization metrics across the New Tab experience.

## Table Details

- **Database**: moz-fx-data-shared-prod
- **Dataset**: telemetry_derived
- **Table**: newtab_clients_daily_v1
- **Partitioning**: Daily by `submission_date` (775 days retention)
- **Clustering**: `channel`, `country_code`
- **Update Schedule**: Daily via `bqetl_newtab_late_morning` DAG
- **Owners**: cbeck@mozilla.com, mbowerman@mozilla.com

## Data Sources

This table aggregates data from:
- `moz-fx-data-shared-prod.telemetry_derived.newtab_visits_v1` (visit-level telemetry)

## Key Metrics & Dimensions

### Client Identifiers
- `client_id`: Primary UUID identifier
- `legacy_telemetry_client_id`: Legacy Firefox telemetry linkage
- `profile_group_id`: Profile group UUID for privacy-preserving analysis

### Configuration & Environment
- Browser metadata (version, name, channel)
- OS details (normalized_os, normalized_os_version)
- Geographic (country_code) and localization (locale)
- Feature flags (Pocket, Top Sites, Weather Widget enablement)

### Engagement Metrics

**Search Engagement**:
- Search counts and ad interactions (clicks, impressions)
- Tagged and follow-on search metrics for attribution

**Content Engagement**:
- **Pocket**: Impressions, clicks, saves, dismissals, thumb voting (sponsored vs organic)
- **Top Sites**: Tile interactions - clicks, impressions, dismissals (sponsored vs organic)
- **List Cards**: New recommendation format with full interaction tracking

**Feature Interactions**:
- **Weather Widget**: Impressions, clicks, display changes, location selections, errors
- **Wallpaper**: Customization clicks, category navigation, highlight interactions
- **Topic Preferences**: Selection, updates, opens, dismissals

### Derived Metrics
- `non_search_engagement_count`: Aggregate non-search interaction count
- `newtab_dismissal_count`: Total dismissal actions across all surfaces
- `visits_with_*`: Visit counts by UI state and engagement type

### Experiments
- RECORD type field capturing active experiments with branch assignments and enrollment details

## ETL Logic

The query performs the following transformations:

1. **Visit Aggregation**: Groups visit-level data by client_id and submission_date
2. **Feature Unnesting**: Expands nested interaction arrays (search, tiles, pocket, weather, etc.)
3. **Boolean Aggregation**: Uses LOGICAL_OR/AND for feature enablement flags
4. **Metric Summation**: Aggregates interaction counts across all visits
5. **COALESCE Handling**: Ensures zero values for clients with no interactions on specific surfaces
6. **Derived Calculations**: Computes composite engagement metrics

## Downstream Analysis Suggestions

### 1. Sponsored Content Performance Analysis
Analyze ROI and engagement rates for sponsored content across different surfaces:
```sql
SELECT
  submission_date,
  country_code,
  channel,
  SUM(sponsored_topsite_tile_clicks) AS sponsored_tile_clicks,
  SUM(sponsored_topsite_tile_impressions) AS sponsored_tile_impressions,
  SAFE_DIVIDE(SUM(sponsored_topsite_tile_clicks), SUM(sponsored_topsite_tile_impressions)) AS tile_ctr,
  SUM(sponsored_pocket_clicks) AS sponsored_pocket_clicks,
  SUM(sponsored_pocket_impressions) AS sponsored_pocket_impressions,
  SAFE_DIVIDE(SUM(sponsored_pocket_clicks), SUM(sponsored_pocket_impressions)) AS pocket_ctr,
  SUM(tagged_search_ad_clicks) AS search_ad_clicks,
  SUM(tagged_search_ad_impressions) AS search_ad_impressions
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY 1, 2, 3
ORDER BY submission_date DESC;
```

### 2. Feature Adoption & Engagement Funnel
Track adoption and active usage of New Tab features over time:
```sql
SELECT
  submission_date,
  COUNT(DISTINCT client_id) AS total_clients,
  COUNT(DISTINCT IF(weather_widget_impressions > 0, client_id, NULL)) AS weather_users,
  COUNT(DISTINCT IF(pocket_clicks > 0, client_id, NULL)) AS pocket_engaged_users,
  COUNT(DISTINCT IF(wallpaper_clicks > 0, client_id, NULL)) AS wallpaper_users,
  COUNT(DISTINCT IF(topic_preferences_set, client_id, NULL)) AS topic_preference_users,
  AVG(non_search_engagement_count) AS avg_non_search_engagements
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY 1
ORDER BY 1 DESC;
```

### 3. Search Monetization Attribution
Analyze search-driven revenue attribution by geography and user segment:
```sql
SELECT
  country_code,
  activity_segment,
  default_search_engine,
  COUNT(DISTINCT client_id) AS clients,
  SUM(searches) AS total_searches,
  SUM(tagged_search_ad_clicks) AS ad_clicks,
  SUM(follow_on_search_ad_clicks) AS follow_on_clicks,
  SAFE_DIVIDE(SUM(tagged_search_ad_clicks), SUM(searches)) AS search_to_ad_rate
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1`
WHERE submission_date BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY) AND CURRENT_DATE()
  AND searches > 0
GROUP BY 1, 2, 3
HAVING clients >= 100
ORDER BY total_searches DESC;
```

### 4. User Sentiment & Content Quality
Evaluate user satisfaction through feedback signals and dismissal patterns:
```sql
SELECT
  submission_date,
  SUM(pocket_thumbs_up) AS thumbs_up,
  SUM(pocket_thumbs_down) AS thumbs_down,
  SAFE_DIVIDE(SUM(pocket_thumbs_up), SUM(pocket_thumb_voting_events)) AS positive_sentiment_rate,
  SUM(sponsored_pocket_dismissals) AS sponsored_dismissals,
  SUM(organic_pocket_dismissals) AS organic_dismissals,
  SUM(topsite_tile_dismissals) AS tile_dismissals,
  AVG(newtab_dismissal_count) AS avg_dismissals_per_client
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_clients_daily_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 60 DAY)
  AND pocket_thumb_voting_events > 0
GROUP BY 1
ORDER BY 1 DESC;
```

## Notes

- **Data Availability**: `topsites_sponsored_tiles_configured` is only available for Firefox 123+ (released 2024-02-20)
- **List Cards**: New content format tracked separately from traditional Pocket cards
- **Experiments**: Use UNNEST on experiments field to analyze A/B test participation
- **Privacy**: profile_group_id is isolated from other telemetry for privacy protection
- **Performance**: Always include submission_date filter due to required partition filtering

## Related Tables

- `telemetry_derived.newtab_visits_v1`: Visit-level source data
- `telemetry.main`: Legacy telemetry linkage via legacy_telemetry_client_id
- `telemetry_derived.clients_daily_v6`: Cross-product client activity analysis
