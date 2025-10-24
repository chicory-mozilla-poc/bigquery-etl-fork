# Newtab Daily Interactions Aggregates

## Overview

This table provides daily aggregated metrics for Firefox newtab interactions, tracking user engagement with top sites, Pocket content, and search functionality. The data is aggregated from `telemetry_derived.newtab_interactions_v1` and partitioned by submission date for efficient querying.

## Data Source

**Source Table:** `moz-fx-data-shared-prod.telemetry_derived.newtab_interactions_v1`

## Key Features

- **Daily Partitioning:** Data is partitioned by `submission_date` for optimal query performance
- **Clustering:** Organized by `channel` and `country_code` for faster filtering
- **Comprehensive Metrics:** Tracks clicks, impressions, dismissals, saves, and searches across multiple content types
- **Sponsored vs Organic:** Separates sponsored and organic interactions for revenue and engagement analysis

## Metric Categories

### 1. User & Configuration Dimensions
- Geographic: country, OS details
- Browser: version, channel, name
- Profile: new profile flag, activity segment
- Settings: search engines, Pocket preferences, topsite configurations

### 2. Top Sites Metrics
- Clicks, impressions, and dismissals
- Breakdown by sponsored vs organic content

### 3. Search Metrics
- Total searches initiated from newtab
- Ad clicks and impressions (tagged and follow-on)
- Search engine and access point tracking

### 4. Pocket Content Metrics
- Story impressions, clicks, and saves
- Position tracking for stories
- Sponsored vs organic content performance

## Downstream Analysis Suggestions

### 1. Sponsored Content Performance Analysis
```sql
-- Calculate click-through rates for sponsored vs organic content
SELECT
  submission_date,
  country_code,
  SUM(sponsored_topsite_clicks) / NULLIF(SUM(sponsored_topsite_impressions), 0) AS sponsored_topsite_ctr,
  SUM(organic_topsite_clicks) / NULLIF(SUM(organic_topsite_impressions), 0) AS organic_topsite_ctr,
  SUM(sponsored_pocket_clicks) / NULLIF(SUM(sponsored_pocket_impressions), 0) AS sponsored_pocket_ctr,
  SUM(organic_pocket_clicks) / NULLIF(SUM(organic_pocket_impressions), 0) AS organic_pocket_ctr
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_daily_interactions_aggregates_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY submission_date, country_code
```

### 2. User Engagement Funnel by Configuration
```sql
-- Analyze engagement patterns based on newtab configuration settings
SELECT
  pocket_enabled,
  topsites_enabled,
  newtab_search_enabled,
  SUM(client_count) AS total_clients,
  SUM(newtab_interactions_visits) / SUM(client_count) AS avg_visits_per_client,
  SUM(searches) / NULLIF(SUM(newtab_interactions_visits), 0) AS search_rate,
  SUM(pocket_clicks) / NULLIF(SUM(newtab_interactions_visits), 0) AS pocket_engagement_rate
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_daily_interactions_aggregates_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
GROUP BY pocket_enabled, topsites_enabled, newtab_search_enabled
```

### 3. New User Adoption and Behavior
```sql
-- Compare behavior between new and existing profiles
SELECT
  is_new_profile,
  channel,
  COUNT(DISTINCT CONCAT(submission_date, country_code, channel)) AS cohort_size,
  AVG(topsite_clicks + pocket_clicks + searches) AS avg_total_interactions,
  SUM(sponsored_topsite_clicks) / NULLIF(SUM(sponsored_topsite_impressions), 0) AS sponsored_ctr,
  SUM(pocket_saves) / NULLIF(SUM(pocket_clicks), 0) AS pocket_save_rate
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_daily_interactions_aggregates_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 14 DAY)
GROUP BY is_new_profile, channel
```

### 4. Search Revenue Attribution Analysis
```sql
-- Analyze search ad performance by engine and access point
SELECT
  search_engine,
  search_access_point,
  country_code,
  SUM(searches) AS total_searches,
  SUM(tagged_search_ad_clicks) AS direct_ad_clicks,
  SUM(follow_on_search_ad_clicks) AS follow_on_ad_clicks,
  SUM(tagged_search_ad_clicks + follow_on_search_ad_clicks) / NULLIF(SUM(searches), 0) AS ads_per_search,
  SUM(tagged_search_ad_clicks) / NULLIF(SUM(tagged_search_ad_impressions), 0) AS direct_ad_ctr
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_daily_interactions_aggregates_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND search_engine IS NOT NULL
GROUP BY search_engine, search_access_point, country_code
```

## Owners

- gkatre@mozilla.com

## Scheduling

- **DAG:** bqetl_newtab
- **Frequency:** Daily
- **Incremental:** Yes

## Schema Details

- **50 columns** including dimensions and metrics
- All columns are NULLABLE
- Data types: DATE, STRING, INTEGER, BOOLEAN

## Notes

- Partition filter on `submission_date` is required for queries
- Metrics are aggregated at the daily level with multiple dimensional breakdowns
- Sponsored content metrics enable revenue and partnership performance tracking
- Activity segment and profile age enable cohort analysis