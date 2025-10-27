# Newtab Conditional Daily Aggregates

## Overview

This table provides daily aggregated metrics for Firefox new tab and homepage interactions, specifically focused on the release channel. It tracks user engagement with various new tab features including top sites, Pocket recommendations, search activity, and different page configurations.

## Data Source

The table aggregates data from `moz-fx-data-shared-prod.telemetry_derived.newtab_interactions_v1`, grouping by submission date, country, and feature enablement flags.

## Key Dimensions

- **submission_date**: Daily partition for time-series analysis
- **country_code**: Geographic segmentation (clustered for query performance)
- **pocket_enabled**: Whether Pocket feature is enabled
- **pocket_sponsored_stories_enabled**: Whether sponsored Pocket stories are enabled
- **topsites_enabled**: Whether top sites feature is enabled

## Metrics Categories

### Total Metrics (All Channels)
- `total_client_count`: All distinct clients
- `total_visit_count`: All distinct new tab visits

### Release Channel Metrics
- `client_count`: Distinct clients on release channel
- `visit_count`: Distinct visits on release channel

### Homepage & New Tab Category Engagement
- Tracks clients and visits with homepage/new tab categories enabled
- Separates about:home and about:newtab behaviors
- Distinguishes default vs. non-default configurations

### Welcome Page Activity
- Tracks visits to about:welcome onboarding page

### Search Behavior
- Counts clients and visits where searches were performed

### Top Sites Engagement
- **Sponsored**: Impressions and clicks on sponsored top sites
- **All**: Impressions and clicks on all top sites (sponsored + organic)

### Pocket Recommendations
- **Sponsored**: Impressions and clicks on sponsored Pocket stories
- **Organic**: Impressions and clicks on organic Pocket recommendations
- **Combined**: Total Pocket impressions and clicks
- **Authentication**: Clients signed in to Pocket

## Table Properties

- **Partitioning**: Daily partitioning by `submission_date` (required for queries)
- **Clustering**: Optimized by `country_code` for geographic analysis
- **Update Schedule**: Daily via `bqetl_newtab` DAG
- **Incremental**: Yes

## Downstream Analysis Suggestions

### 1. Feature Adoption Analysis
Analyze the relationship between feature enablement (Pocket, sponsored stories, top sites) and user engagement rates. Calculate click-through rates (CTR) for sponsored vs. organic content:

```sql
SELECT 
  submission_date,
  pocket_enabled,
  pocket_sponsored_stories_enabled,
  SUM(sponsored_pocket_clicked_visit_count) / NULLIF(SUM(sponsored_pocket_impressed_visit_count), 0) AS sponsored_ctr,
  SUM(organic_pocket_clicked_visit_count) / NULLIF(SUM(organic_pocket_impressed_visit_count), 0) AS organic_ctr
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_conditional_daily_aggregates_v1`
GROUP BY 1, 2, 3
ORDER BY submission_date DESC
```

### 2. Geographic Engagement Patterns
Identify countries with highest engagement rates and compare default vs. non-default configuration adoption:

```sql
SELECT 
  country_code,
  SUM(home_default_client_count) AS default_home_clients,
  SUM(home_nondefault_client_count) AS nondefault_home_clients,
  SUM(home_nondefault_client_count) / NULLIF(SUM(client_count), 0) AS customization_rate
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_conditional_daily_aggregates_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY country_code
ORDER BY default_home_clients DESC
```

### 3. User Journey Analysis
Track the conversion funnel from impressions to clicks across different content types:

```sql
SELECT 
  submission_date,
  SUM(topsite_impressed_visit_count) AS topsite_impressions,
  SUM(topsite_clicked_visit_count) AS topsite_clicks,
  SUM(pocket_impressed_visit_count) AS pocket_impressions,
  SUM(pocket_clicked_visit_count) AS pocket_clicks,
  SUM(searched_visit_count) AS search_visits
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_conditional_daily_aggregates_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY submission_date
ORDER BY submission_date
```

### 4. Welcome Page Impact Assessment
Evaluate how welcome page exposure correlates with subsequent engagement and feature adoption:

```sql
SELECT 
  CASE WHEN welcome_visit_count > 0 THEN 'Visited Welcome' ELSE 'No Welcome Visit' END AS welcome_exposure,
  AVG(searched_visit_count / NULLIF(visit_count, 0)) AS avg_search_rate,
  AVG(topsite_clicked_visit_count / NULLIF(visit_count, 0)) AS avg_topsite_engagement,
  AVG(pocket_clicked_visit_count / NULLIF(visit_count, 0)) AS avg_pocket_engagement
FROM `moz-fx-data-shared-prod.telemetry_derived.newtab_conditional_daily_aggregates_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
GROUP BY welcome_exposure
```

## Owner

- **Primary**: gkatre@mozilla.com
- **Team**: Firefox Desktop
- **DAG**: bqetl_newtab
