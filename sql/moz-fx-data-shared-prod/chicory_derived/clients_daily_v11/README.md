# Newtab Clients Daily (clients_daily_v11)

## Overview

The `clients_daily_v11` table provides a comprehensive daily aggregation of Firefox desktop new tab engagement metrics at the client level. This table consolidates data from the `newtab_visits_daily_v2` source table, summarizing all new tab page visits and interactions for each client on a daily basis.

## Data Source

- **Source Table**: `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`
- **Granularity**: One row per client per day
- **Partitioning**: Daily, by `submission_date` field
- **Clustering**: By `channel`, `country`, and `newtab_category`

## Update Schedule

This table is updated **daily** as part of the `bqetl_newtab` DAG. Data is partitioned by `submission_date` with a retention period of 775 days (~2 years).

## Key Columns

### Identifiers and Dimensions
- **submission_date**: The date the data was submitted (partition field)
- **client_id**: Unique identifier for each Firefox client
- **app_version**, **os**, **channel**, **locale**, **country**: Client configuration and demographic attributes
- **sample_id**, **profile_group_id**: Sampling and grouping identifiers

### Configuration Settings
- **organic_content_enabled**: Whether organic content recommendations are enabled
- **sponsored_content_enabled**: Whether sponsored content is enabled
- **sponsored_topsites_enabled**: Whether sponsored top sites are enabled
- **newtab_search_enabled**: Whether search functionality is enabled in new tab
- **newtab_weather_enabled**: Whether weather widget is enabled

### Visit Metrics
- **all_visits**: Total number of distinct new tab visits
- **default_ui_visits**: Visits where the default UI was displayed
- **any_engagement_visits**: Visits with any user interaction
- **search_engagement_visits**: Visits with search interactions
- **nonsearch_engagement_visits**: Visits with non-search interactions

### Content Engagement Metrics
These metrics track interactions with recommended content (Pocket articles, etc.):
- **organic_content_click_count**: Clicks on organic content recommendations
- **sponsored_content_click_count**: Clicks on sponsored content
- **organic_content_impression_count**: Impressions of organic content
- **sponsored_content_impression_count**: Impressions of sponsored content

### Top Sites Metrics
Metrics for the pinned/frequent top sites tiles:
- **organic_topsite_click_count**: Clicks on organic top sites
- **sponsored_topsite_click_count**: Clicks on sponsored top sites
- **organic_topsite_impression_count**: Impressions of organic top sites
- **sponsored_topsite_impression_count**: Impressions of sponsored top sites

### User Feedback Metrics
- **content_thumbs_up_count**: Positive feedback on content
- **content_thumbs_down_count**: Negative feedback on content
- **organic_content_dismissal_count**: Times organic content was dismissed
- **sponsored_content_dismissal_count**: Times sponsored content was dismissed

### Search Advertising Metrics
- **search_ad_click_count**: Clicks on search ads from new tab
- **search_ad_impression_count**: Impressions of search ads

### Duration Metrics
- **avg_newtab_visit_duration**: Average duration per new tab visit (seconds)
- **cumulative_newtab_visit_duration**: Total time spent across all new tab visits

## Example Use Cases

### 1. **Sponsored Content Performance Analysis**
Analyze the effectiveness of sponsored content by comparing click-through rates across different client segments:
```sql
SELECT
  country,
  channel,
  COUNT(DISTINCT client_id) AS total_clients,
  SUM(sponsored_content_click_count) AS total_clicks,
  SUM(sponsored_content_impression_count) AS total_impressions,
  SAFE_DIVIDE(SUM(sponsored_content_click_count), SUM(sponsored_content_impression_count)) AS ctr
FROM
  `moz-fx-data-shared-prod.chicory_derived.clients_daily_v11`
WHERE
  submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
  AND sponsored_content_enabled = TRUE
GROUP BY
  country, channel
ORDER BY
  total_clients DESC
```

### 2. **New Tab Engagement Trends**
Track daily engagement trends to understand how users interact with the new tab page:
```sql
SELECT
  submission_date,
  COUNT(DISTINCT client_id) AS active_clients,
  AVG(all_visits) AS avg_visits_per_client,
  AVG(any_engagement_visits) AS avg_engaged_visits,
  SAFE_DIVIDE(SUM(any_engagement_visits), SUM(all_visits)) AS engagement_rate,
  AVG(avg_newtab_visit_duration) AS avg_visit_duration_sec
FROM
  `moz-fx-data-shared-prod.chicory_derived.clients_daily_v11`
WHERE
  submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY
  submission_date
ORDER BY
  submission_date
```

### 3. **Content Recommendation Performance by Geography**
Compare organic vs. sponsored content performance across different regions:
```sql
SELECT
  country,
  SUM(organic_content_engagement_visits) AS organic_engagements,
  SUM(sponsored_content_engagement_visits) AS sponsored_engagements,
  SUM(organic_content_click_count) AS organic_clicks,
  SUM(sponsored_content_click_count) AS sponsored_clicks,
  SUM(organic_content_dismissal_count) AS organic_dismissals,
  SUM(sponsored_content_dismissal_count) AS sponsored_dismissals
FROM
  `moz-fx-data-shared-prod.chicory_derived.clients_daily_v11`
WHERE
  submission_date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
GROUP BY
  country
HAVING
  SUM(all_visits) > 1000
ORDER BY
  organic_clicks + sponsored_clicks DESC
LIMIT 20
```

### 4. **User Feedback and Content Quality Analysis**
Analyze user sentiment towards content recommendations:
```sql
SELECT
  newtab_category,
  COUNT(DISTINCT client_id) AS clients,
  SUM(content_thumbs_up_count) AS positive_feedback,
  SUM(content_thumbs_down_count) AS negative_feedback,
  SAFE_DIVIDE(SUM(content_thumbs_up_count),
    SUM(content_thumbs_up_count) + SUM(content_thumbs_down_count)) AS positive_ratio,
  AVG(any_content_click_count) AS avg_content_clicks
FROM
  `moz-fx-data-shared-prod.chicory_derived.clients_daily_v11`
WHERE
  submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
  AND (content_thumbs_up_count > 0 OR content_thumbs_down_count > 0)
GROUP BY
  newtab_category
ORDER BY
  clients DESC
```

## Notes

- This table uses the `mode_last` UDF to select the most frequently occurring value for configuration fields when aggregating multiple visits in a day
- Fields added in August 2025 include enhanced experiment tracking, weather widget metrics, and additional user feedback data
- The table is partitioned and clustered for optimal query performance when filtering by date, channel, country, or newtab_category
- All count and sum metrics are nullable and will be NULL if no data is available for that metric
