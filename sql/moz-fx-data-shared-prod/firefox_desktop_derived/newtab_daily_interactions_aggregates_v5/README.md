# Firefox Desktop New Tab Daily Interactions Aggregates v5

## Overview

This table provides daily aggregated metrics for Firefox Desktop new tab page interactions. It consolidates user engagement data across various content types including organic and sponsored content, topsites, widgets, and search functionality. The data is aggregated from the `newtab_clients_daily_v2` table and grouped by key dimensions to enable comprehensive analysis of new tab page performance and user behavior.

## Table Details

- **Dataset**: `moz-fx-data-shared-prod.firefox_desktop_derived`
- **Table**: `newtab_daily_interactions_aggregates_v5`
- **Update Frequency**: Daily
- **Partitioning**: By `submission_date`
- **Source Table**: `firefox_desktop_derived.newtab_clients_daily_v2`

## Schema Overview

### Dimensions

The table aggregates data across the following dimensions:
- **Date & Version**: submission_date, app_version
- **Client Context**: os, channel, locale, country
- **Configuration**: homepage_category, newtab_category
- **Feature Flags**: organic_content_enabled, sponsored_content_enabled, sponsored_topsites_enabled, organic_topsites_enabled, newtab_search_enabled

### Metrics

Metrics are organized into several categories:

1. **Overall Engagement**
   - Total visits and unique clients
   - Default UI interactions
   - General engagement metrics

2. **Content Engagement** (Organic & Sponsored)
   - Visits, clients, clicks, and impressions
   - Separated by organic and sponsored content

3. **Topsite Engagement** (Organic & Sponsored)
   - Visits, clients, clicks, and impressions
   - Separated by organic and sponsored topsites

4. **User Feedback Metrics** (Added September 2025)
   - Content and topsite dismissals
   - Thumbs up/down interactions

5. **Search Metrics**
   - Search engagement, ad clicks, and impressions
   - Search interaction counts

6. **Widget & Other Engagement**
   - Widget interactions and impressions
   - Other engagement types

## ETL Logic

The table is built using aggregation queries that:
1. Filter data for a specific submission_date using a parameterized query (`@submission_date`)
2. Sum numerical metrics across all clients
3. Count distinct clients for engagement-based metrics using conditional logic
4. Group by all dimension columns to create granular aggregates

## Downstream Analysis Suggestions

### 1. Content Performance Analysis
Analyze the effectiveness of organic vs. sponsored content by comparing click-through rates (CTR) and engagement rates across different dimensions:
```sql
SELECT
  country,
  channel,
  organic_content_click_count / NULLIF(organic_content_impression_count, 0) AS organic_ctr,
  sponsored_content_click_count / NULLIF(sponsored_content_impression_count, 0) AS sponsored_ctr,
  organic_content_engagement_clients,
  sponsored_content_engagement_clients
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_daily_interactions_aggregates_v5`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY country, channel, organic_ctr, sponsored_ctr, organic_content_engagement_clients, sponsored_content_engagement_clients
ORDER BY organic_content_engagement_clients DESC
```

### 2. User Feedback Sentiment Tracking
Monitor user sentiment through dismissal rates and thumbs up/down ratios to identify content quality issues:
```sql
SELECT
  submission_date,
  locale,
  (content_thumbs_up_count / NULLIF(content_thumbs_up_count + content_thumbs_down_count, 0)) AS positive_feedback_ratio,
  (organic_content_dismissal_count + sponsored_content_dismissal_count) AS total_content_dismissals,
  any_content_impression_count
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_daily_interactions_aggregates_v5`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
ORDER BY submission_date DESC
```

### 3. Feature Adoption and Impact Analysis
Evaluate how different feature flag combinations affect user engagement:
```sql
SELECT
  organic_content_enabled,
  sponsored_content_enabled,
  sponsored_topsites_enabled,
  SUM(any_engagement_clients) AS total_engaged_clients,
  SUM(any_engagement_visits) / NULLIF(SUM(all_visits), 0) AS engagement_rate,
  AVG(search_ad_click_count / NULLIF(search_ad_impression_count, 0)) AS avg_search_ad_ctr
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_daily_interactions_aggregates_v5`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
GROUP BY organic_content_enabled, sponsored_content_enabled, sponsored_topsites_enabled
ORDER BY total_engaged_clients DESC
```

### 4. Geographic and Channel Performance Comparison
Compare new tab engagement across different markets and release channels to identify optimization opportunities:
```sql
SELECT
  country,
  channel,
  SUM(newtab_clients) AS total_clients,
  SUM(nonsearch_engagement_visits) / NULLIF(SUM(all_visits), 0) AS nonsearch_engagement_rate,
  SUM(widget_engagement_visits) / NULLIF(SUM(all_visits), 0) AS widget_engagement_rate,
  SUM(any_topsite_click_count) / NULLIF(SUM(any_topsite_impression_count), 0) AS topsite_ctr
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_daily_interactions_aggregates_v5`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY country, channel
HAVING total_clients > 1000
ORDER BY total_clients DESC
```

## Owners

- **Team**: Firefox Desktop Experience
- **Contact**: (To be determined from metadata.yaml)

## Notes

- New dismissal and feedback metrics were added in September 2025
- The table uses parameterized queries for efficient daily updates
- All client counts use DISTINCT to avoid double-counting
- Engagement metrics are conditional to only count clients with activity > 0

## Related Tables

- **Source**: `firefox_desktop_derived.newtab_clients_daily_v2`
- **Related**: Other newtab interaction tables in the firefox_desktop_derived dataset
