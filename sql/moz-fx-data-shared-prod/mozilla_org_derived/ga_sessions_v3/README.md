# Google Analytics Sessions V3

## Table Overview

The `ga_sessions_v3` table provides session-level analytics data from Google Analytics 4 for mozilla.org web properties. Each row represents a unique session identified by the composite key (`ga_client_id`, `ga_session_id`). This table aggregates event-level data into session summaries, capturing user behavior, traffic attribution, device characteristics, and conversion events.

## Data Source

Data is sourced from `analytics_313696158.events_*` in the marketing production project. The ETL process uses a MERGE strategy that:
- Scans for sessions with events in the last 3 days plus the submission date
- Recalculates session-level metrics from all historical events for those sessions
- Upserts session records to maintain the most current view of each session

## Key Features

- **Incremental Updates**: Sessions are updated when new events arrive, ensuring recent data accuracy
- **Partitioning**: Time-partitioned by `session_date` with 775-day retention
- **Clustering**: Clustered by `ga_client_id` and `country` for efficient query performance
- **Attribution Tracking**: Comprehensive traffic source tracking including manual UTM parameters, Google Ads data, and cross-channel campaigns
- **Conversion Tracking**: Captures Firefox download events, install targets, and stub session IDs for download attribution
- **Experiment Data**: Tracks A/B test participation via experiment IDs and branches

## Schema Highlights

### Identity & Session Keys
- `ga_client_id`: Cookie-based client identifier
- `ga_session_id`: Unique session identifier
- `session_number`: Sequential counter of sessions per client
- `is_first_session`: New vs. returning user flag

### Engagement Metrics
- `time_on_site`: Session duration in seconds
- `pageviews`: Total page views
- `session_start_timestamp`: Session start time

### Device & Location Context
- Device attributes: `device_category`, `mobile_device_model`, `os`, `browser`
- Geographic data: `country`, `region`, `city`
- Language preference: `language`

### Attribution Sources

**Manual/UTM Parameters** (from `collected_traffic_source`):
- Campaign: `manual_campaign_id`, `manual_campaign_name`
- Traffic: `manual_source`, `manual_medium`, `manual_term`, `manual_content`

**Google Ads** (from `session_traffic_source_last_click`):
- Campaign: `ad_google_campaign`, `ad_google_campaign_id`
- Ad Group: `ad_google_group`, `ad_google_group_id`
- Account: `ad_google_account`
- Click IDs: `gclid`, `gclid_array`

**Cross-Channel Campaigns**:
- `ad_crosschannel_source`, `ad_crosschannel_medium`, `ad_crosschannel_campaign`
- Channel groupings: `ad_crosschannel_primary_channel_group`, `ad_crosschannel_default_channel_group`

**Event Parameter Attribution** (first value and all distinct values):
- Campaign, source, medium, content, term
- Google Ads campaign IDs
- Experiment IDs and branches

### Conversion Tracking
- `had_download_event`: Download occurrence flag
- `firefox_desktop_downloads`: Desktop download count
- `last_reported_install_target`, `all_reported_install_targets`: Product/platform installed
- `last_reported_stub_session_id`, `all_reported_stub_session_ids`: Links to download tokens
- `landing_screen`: Entry page URL

## Downstream Analysis Suggestions

### 1. Campaign Attribution Analysis
Analyze which marketing campaigns drive the most engaged sessions and conversions:

```sql
SELECT 
  COALESCE(manual_campaign_name, first_campaign_from_event_params, 'Direct') AS campaign,
  COALESCE(manual_source, first_source_from_event_params) AS source,
  COALESCE(manual_medium, first_medium_from_event_params) AS medium,
  COUNT(DISTINCT ga_session_id) AS sessions,
  COUNT(DISTINCT ga_client_id) AS unique_users,
  AVG(time_on_site) AS avg_session_duration,
  AVG(pageviews) AS avg_pageviews,
  COUNTIF(had_download_event) AS downloads,
  SAFE_DIVIDE(COUNTIF(had_download_event), COUNT(DISTINCT ga_session_id)) AS download_rate
FROM `moz-fx-data-shared-prod.mozilla_org_derived.ga_sessions_v3`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY campaign, source, medium
ORDER BY sessions DESC;
```

### 2. User Journey & Session Sequence Analysis
Understand user behavior across multiple sessions from first visit to conversion:

```sql
SELECT 
  session_number,
  COUNT(DISTINCT ga_client_id) AS users_at_session_n,
  AVG(time_on_site) AS avg_duration,
  AVG(pageviews) AS avg_pageviews,
  COUNTIF(had_download_event) AS downloads,
  COUNTIF(is_first_session) AS new_users
FROM `moz-fx-data-shared-prod.mozilla_org_derived.ga_sessions_v3`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  AND session_number <= 10
GROUP BY session_number
ORDER BY session_number;
```

### 3. Google Ads Performance Tracking
Evaluate Google Ads campaign effectiveness with detailed ad group and cross-channel metrics:

```sql
SELECT 
  ad_google_campaign,
  ad_google_group,
  ad_crosschannel_default_channel_group,
  COUNT(DISTINCT ga_session_id) AS sessions,
  COUNT(DISTINCT CASE WHEN is_first_session THEN ga_client_id END) AS new_users,
  SUM(firefox_desktop_downloads) AS total_downloads,
  SAFE_DIVIDE(SUM(firefox_desktop_downloads), COUNT(DISTINCT ga_session_id)) AS downloads_per_session,
  AVG(time_on_site) AS avg_session_duration,
  COUNT(DISTINCT gclid) AS unique_clicks
FROM `moz-fx-data-shared-prod.mozilla_org_derived.ga_sessions_v3`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND ad_google_campaign IS NOT NULL
GROUP BY ad_google_campaign, ad_google_group, ad_crosschannel_default_channel_group
ORDER BY sessions DESC;
```

### 4. A/B Test Experiment Analysis
Measure the impact of experiments on user engagement and conversion rates:

```sql
SELECT 
  first_experiment_id_from_event_params AS experiment_id,
  first_experiment_branch_from_event_params AS branch,
  COUNT(DISTINCT ga_session_id) AS sessions,
  COUNT(DISTINCT ga_client_id) AS unique_users,
  AVG(pageviews) AS avg_pageviews,
  AVG(time_on_site) AS avg_time_on_site,
  COUNTIF(had_download_event) AS downloads,
  SAFE_DIVIDE(COUNTIF(had_download_event), COUNT(DISTINCT ga_session_id)) AS conversion_rate
FROM `moz-fx-data-shared-prod.mozilla_org_derived.ga_sessions_v3`
WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND first_experiment_id_from_event_params IS NOT NULL
GROUP BY experiment_id, branch
ORDER BY experiment_id, branch;
```

## Notes

- Sessions without a `ga_session_id` are excluded from this table
- The ETL looks back 3 days to capture late-arriving events and update existing sessions
- Session attributes prioritize `session_start` event data when available, falling back to first event in session
- Attribution arrays (e.g., `distinct_campaigns_from_event_params`) capture all unique values, useful for multi-touch attribution
- Stub session IDs enable joining with download token tables for full funnel analysis
- Time partitioning with 775-day retention ensures compliance with data retention policies