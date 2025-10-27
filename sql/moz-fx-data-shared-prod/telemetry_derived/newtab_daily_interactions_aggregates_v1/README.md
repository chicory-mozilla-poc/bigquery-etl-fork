# Newtab Daily Interactions Aggregates

## Overview

This table provides daily aggregated metrics of user interactions with the Firefox newtab page, including engagement with top sites, Pocket recommendations, and search functionality. The data is aggregated from newtab_interactions_v1 and partitioned by day for efficient querying.

## Data Source

Source Table: moz-fx-data-shared-prod.telemetry_derived.newtab_interactions_v1

## Key Features

- Daily Aggregation: Summarizes newtab interaction events by submission date
- Multi-Dimensional: Segmented by country, channel, browser version, OS, search engine, and user preferences
- Content Types: Tracks three main content areas - Top Sites, Pocket, and Search
- Sponsored vs Organic: Separates metrics for sponsored and organic content
- Time Partitioning: Partitioned by submission_date with required partition filter
- Clustering: Optimized for queries filtering by channel and country_code

## Schema Structure

### Dimension Fields
- User context: browser version, OS, country, channel, activity segment
- Configuration settings: Pocket enabled, top sites rows, search enabled
- Search context: search engine, access point, default engines
- Pocket settings: signed in status, sponsored stories enabled

### Metric Fields

Client Counts:
- client_count: Unique clients with interactions
- legacy_telemetry_client_count: Clients using legacy telemetry
- newtab_interactions_visits: Distinct newtab visits

Top Sites Metrics:
- Clicks, impressions, and dismissals (total, sponsored, organic)

Search Metrics:
- Search queries, ad clicks, ad impressions
- Tagged and follow-on search ad engagement

Pocket Metrics:
- Story impressions, clicks, and saves
- Separated by sponsored and organic content

## Downstream Analysis Suggestions

### 1. Sponsored Content Performance Analysis
Analyze the effectiveness of sponsored content across different regions and user segments:
sql
SELECT 
 country_code,
 activity_segment,
 SUM(sponsored_topsite_impressions) as sponsored_impressions,
 SUM(sponsored_topsite_clicks) as sponsored_clicks,
 SAFE_DIVIDE(SUM(sponsored_topsite_clicks), SUM(sponsored_topsite_impressions)) as ctr
FROM moz-fx-data-shared-prod.telemetry_derived.newtab_daily_interactions_aggregates_v1
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY country_code, activity_segment
ORDER BY sponsored_impressions DESC;


### 2. Search Monetization Funnel
Track the search-to-ad-click funnel to understand monetization efficiency:
sql
SELECT 
 submission_date,
 search_engine,
 SUM(searches) as total_searches,
 SUM(tagged_search_ad_impressions) as ad_impressions,
 SUM(tagged_search_ad_clicks) as ad_clicks,
 SAFE_DIVIDE(SUM(tagged_search_ad_clicks), SUM(tagged_search_ad_impressions)) as ad_ctr
FROM moz-fx-data-shared-prod.telemetry_derived.newtab_daily_interactions_aggregates_v1
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY submission_date, search_engine
ORDER BY submission_date DESC;


### 3. Feature Adoption and Engagement
Measure adoption rates of newtab features and their impact on engagement:
sql
SELECT 
 pocket_enabled,
 topsites_enabled,
 newtab_search_enabled,
 SUM(client_count) as clients,
 AVG(SAFE_DIVIDE(topsite_clicks, NULLIF(client_count, 0))) as avg_topsite_clicks_per_client,
 AVG(SAFE_DIVIDE(pocket_clicks, NULLIF(client_count, 0))) as avg_pocket_clicks_per_client,
 AVG(SAFE_DIVIDE(searches, NULLIF(client_count, 0))) as avg_searches_per_client
FROM moz-fx-data-shared-prod.telemetry_derived.newtab_daily_interactions_aggregates_v1
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY pocket_enabled, topsites_enabled, newtab_search_enabled
ORDER BY clients DESC;


### 4. New vs Existing Profile Behavior
Compare engagement patterns between new and existing Firefox profiles:
sql
SELECT 
 is_new_profile,
 channel,
 SUM(client_count) as total_clients,
 SAFE_DIVIDE(SUM(topsite_clicks), SUM(client_count)) as avg_topsite_clicks,
 SAFE_DIVIDE(SUM(pocket_clicks), SUM(client_count)) as avg_pocket_clicks,
 SAFE_DIVIDE(SUM(searches), SUM(client_count)) as avg_searches,
 SAFE_DIVIDE(SUM(sponsored_topsite_dismissals), NULLIF(SUM(sponsored_topsite_impressions), 0)) as sponsored_dismissal_rate
FROM moz-fx-data-shared-prod.telemetry_derived.newtab_daily_interactions_aggregates_v1
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY is_new_profile, channel
ORDER BY total_clients DESC;


## Scheduling

- DAG: bqetl_newtab
- Schedule: Daily
- Owner: [gkatre@mozilla.com](mailto:gkatre@mozilla.com)

## Notes

- All metrics are pre-aggregated sums from the base newtab_interactions_v1 table
- Use partition filters on submission_date for optimal query performance
- Clustering on channel and country_code improves query efficiency for these common filters
- NULL values in sponsored/organic metrics indicate no activity of that type
