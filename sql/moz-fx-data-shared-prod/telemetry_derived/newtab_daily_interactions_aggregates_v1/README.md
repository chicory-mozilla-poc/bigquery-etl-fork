# Newtab Daily Interactions Aggregates

## Overview

This table provides daily aggregated metrics of user interactions with Firefox's New Tab page, including engagement with topsites, Pocket recommendations, and search features. The data is aggregated from the `newtab_interactions_v1` source table and partitioned by submission date.

## Data Characteristics

- **Granularity**: Daily aggregations grouped by multiple dimensions
- **Partitioning**: Day-level partitioning on `submission_date` (partition filter required)
- **Clustering**: Optimized for queries filtering by `channel` and `country_code`
- **Update Schedule**: Daily via `bqetl_newtab` DAG
- **Owner**: gkatre@mozilla.com

## Key Metrics

### User Counts
- **client_count**: Distinct clients with new tab interactions
- **legacy_telemetry_client_count**: Clients using legacy telemetry
- **newtab_interactions_visits**: Distinct new tab visit sessions

### Topsite Engagement
- Clicks, impressions, and dismissals tracked separately for sponsored and organic topsites
- Measures user interaction with frequently visited sites displayed on new tab

### Search Metrics
- Total searches initiated from new tab
- Ad clicks and impressions with attribution tracking
- Follow-on search ad engagement metrics

### Pocket Integration
- Impressions, clicks, and saves for Pocket story recommendations
- Separate tracking for sponsored vs. organic content
- Position-based analysis support

## Dimensional Attributes

The table supports analysis across multiple dimensions:
- **Geographic**: country_code
- **Browser**: channel, browser_version, browser_name, os, os_version
- **Configuration**: search engines, Pocket settings, topsite settings
- **User Segments**: activity_segment, is_new_profile

## Downstream Analysis Suggestions

### 1. Sponsored Content Performance Analysis
Analyze the effectiveness of sponsored content by comparing click-through rates (CTR) and engagement rates between sponsored and organic content across Pocket stories and topsites.

```sql
SELECT
  submission_date,
  country_code,
  SAFE_DIVIDE(sponsored_pocket_clicks, sponsored_pocket_impressions) AS sponsored_pocket_ctr,
  SAFE_DIVIDE(organic_pocket_clicks, organic_pocket_impressions) AS organic_pocket_ctr,
  SAFE_DIVIDE(sponsored_topsite_clicks, sponsored_topsite_impressions) AS sponsored_topsite_ctr,
  SAFE_DIVIDE(organic_topsite_clicks, organic_topsite_impressions) AS organic_topsite_ctr
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_daily_interactions_aggregates_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY submission_date, country_code
ORDER BY submission_date DESC;
```

### 2. Search Monetization Funnel
Track the search-to-ad-click conversion funnel to understand monetization effectiveness across different search engines and access points.

```sql
SELECT
  search_engine,
  search_access_point,
  SUM(searches) AS total_searches,
  SUM(tagged_search_ad_impressions) AS ad_impressions,
  SUM(tagged_search_ad_clicks) AS ad_clicks,
  SAFE_DIVIDE(SUM(tagged_search_ad_clicks), SUM(searches)) AS search_to_click_rate,
  SAFE_DIVIDE(SUM(tagged_search_ad_clicks), SUM(tagged_search_ad_impressions)) AS ad_ctr
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_daily_interactions_aggregates_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  AND search_engine IS NOT NULL
GROUP BY search_engine, search_access_point
ORDER BY total_searches DESC;
```

### 3. New User Onboarding and Feature Adoption
Compare engagement patterns between new and existing profiles to optimize onboarding experiences and feature discovery.

```sql
SELECT
  is_new_profile,
  activity_segment,
  AVG(SAFE_DIVIDE(topsite_clicks, client_count)) AS avg_topsite_clicks_per_user,
  AVG(SAFE_DIVIDE(searches, client_count)) AS avg_searches_per_user,
  AVG(SAFE_DIVIDE(pocket_clicks, client_count)) AS avg_pocket_clicks_per_user,
  SUM(client_count) AS total_clients
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_daily_interactions_aggregates_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
GROUP BY is_new_profile, activity_segment
ORDER BY is_new_profile, total_clients DESC;
```

### 4. Feature Configuration Impact Analysis
Evaluate how different new tab page configurations (Pocket enabled/disabled, topsite rows, search enabled) impact user engagement metrics.

```sql
SELECT
  pocket_enabled,
  pocket_sponsored_stories_enabled,
  topsites_enabled,
  newtab_search_enabled,
  topsites_rows,
  SUM(client_count) AS users,
  SAFE_DIVIDE(SUM(newtab_interactions_visits), SUM(client_count)) AS avg_visits_per_user,
  SAFE_DIVIDE(SUM(topsite_clicks + pocket_clicks + searches), SUM(client_count)) AS avg_interactions_per_user
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_daily_interactions_aggregates_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY pocket_enabled, pocket_sponsored_stories_enabled, topsites_enabled, newtab_search_enabled, topsites_rows
HAVING users > 1000
ORDER BY users DESC;
```

## Usage Notes

- Always include `submission_date` filter to avoid full table scan (partition filter required)
- Use clustering columns (`channel`, `country_code`) in WHERE clauses for optimal performance
- NULL values in dimensional columns represent uncategorized or missing data
- All metric columns are aggregated sums and should be further aggregated for multi-day analysis
- This table is incrementally updated daily and may not contain the most recent day's data

## Related Tables

- **Source**: `moz-fx-data-shared-prod.telemetry_derived.newtab_interactions_v1`
- **DAG**: bqetl_newtab