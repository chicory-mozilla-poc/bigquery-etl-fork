# GA Sessions V3

## Table Overview

The `ga_sessions_v3` table contains session-level aggregated data from Google Analytics 4 (GA4) events for mozilla.org. Each row represents a unique user session, identified by the composite primary key of `ga_client_id` and `ga_session_id`.

### Key Characteristics

- **Grain**: One row per GA4 session
- **Primary Key**: (`ga_client_id`, `ga_session_id`)
- **Update Pattern**: Incremental MERGE operation that upserts sessions with events from the last 3 days
- **Partitioning**: Day-partitioned on `session_date` (775-day expiration)
- **Clustering**: Clustered on `ga_client_id` and `country` for query optimization

## Data Sources

This table is derived from:
- `moz-fx-data-marketing-prod.analytics_313696158.events_*` - Raw GA4 event data

## ETL Logic

The script performs incremental processing:
1. Identifies all unique (ga_client_id, ga_session_id) combinations with events in the last 3 days
2. For each session, re-aggregates all historical events for that session
3. Extracts session attributes from the `session_start` event (device, location, browser, campaign data)
4. Aggregates event-level parameters (campaigns, sources, mediums) using first-value and distinct-array patterns
5. Calculates session metrics (pageviews, time on site, download events)
6. Performs MERGE operation to insert new sessions or update existing ones

### Attribution Data

The table captures attribution from multiple sources:
- **Manual attribution**: From `collected_traffic_source` (manual_campaign_id, manual_source, manual_medium, etc.)
- **Event parameters**: First and distinct values for campaign, source, medium, content, term
- **Google Ads**: Campaign, group, and account data from `session_traffic_source_last_click`
- **Cross-channel**: Multi-channel attribution data including source platform and channel groupings

## Downstream Analysis Suggestions

### 1. User Acquisition Analysis
```sql
-- Analyze session acquisition by channel and campaign
SELECT 
  manual_medium,
  manual_campaign_name,
  COUNT(DISTINCT ga_client_id) as unique_users,
  COUNT(*) as total_sessions,
  SUM(firefox_desktop_downloads) as total_downloads,
  SAFE_DIVIDE(SUM(firefox_desktop_downloads), COUNT(*)) as download_rate
FROM `moz-fx-data-shared-prod.mozilla_org_derived.ga_sessions_v3`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND manual_medium IS NOT NULL
GROUP BY manual_medium, manual_campaign_name
ORDER BY total_sessions DESC;
```

### 2. Session Engagement Metrics
```sql
-- Calculate engagement metrics by device category
SELECT 
  device_category,
  COUNT(*) as sessions,
  AVG(time_on_site) as avg_time_on_site_seconds,
  AVG(pageviews) as avg_pageviews,
  COUNTIF(is_first_session) as first_sessions,
  SAFE_DIVIDE(COUNTIF(is_first_session), COUNT(*)) as first_session_rate
FROM `moz-fx-data-shared-prod.mozilla_org_derived.ga_sessions_v3`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
GROUP BY device_category;
```

### 3. Geographic Distribution Analysis
```sql
-- Analyze user sessions and downloads by geography
SELECT 
  country,
  region,
  COUNT(DISTINCT ga_client_id) as unique_clients,
  COUNT(*) as total_sessions,
  SUM(firefox_desktop_downloads) as downloads,
  COUNTIF(had_download_event) as sessions_with_downloads
FROM `moz-fx-data-shared-prod.mozilla_org_derived.ga_sessions_v3`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY country, region
ORDER BY total_sessions DESC
LIMIT 100;
```

### 4. Google Ads Campaign Performance
```sql
-- Evaluate Google Ads campaign effectiveness
SELECT 
  ad_google_campaign,
  ad_google_group,
  COUNT(*) as sessions,
  COUNT(DISTINCT ga_client_id) as unique_users,
  SUM(firefox_desktop_downloads) as downloads,
  SUM(pageviews) as total_pageviews,
  AVG(time_on_site) as avg_time_on_site
FROM `moz-fx-data-shared-prod.mozilla_org_derived.ga_sessions_v3`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND ad_google_campaign IS NOT NULL
GROUP BY ad_google_campaign, ad_google_group
ORDER BY sessions DESC;
```

## Important Notes

- Sessions without a non-null `ga_session_id` are excluded from this table
- The table looks back at all historical events when recalculating sessions, not just the last 3 days
- Country fallback logic: uses session_start geo.country, or first reported country from event_params if no session_start event
- Timestamp conversion uses "Europe/London" timezone
- Download events changed naming convention on 2024-02-17 (from `product_download` to `firefox_download`)

## Owner

- Primary: kwindau@mozilla.com