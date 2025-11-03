# Firefox Desktop Newtab Visits Daily (v2)

## Table Overview

This table provides daily-aggregated metrics for Firefox Desktop newtab page visits, with one row per unique visit (identified by `newtab_visit_id`) per day. It captures comprehensive user interaction data including content clicks, impressions, search behavior, and widget engagement on the Firefox newtab page.

**Key Information:**
- **Owner:** gkatre@mozilla.com
- **Schedule:** Daily (DAG: bqetl_newtab)
- **Partitioning:** Daily by `submission_date` (required filter)
- **Clustering:** By `channel`, `country`, `newtab_category`
- **Expiration:** 775 days
- **Granularity:** One row per newtab visit per day

## Data Source

The table aggregates event-level data from the `firefox_desktop_stable.newtab_v1` ping, processing events related to:
- Newtab page opens and closes
- Search interactions and ad clicks
- Pocket content (clicks, impressions, dismissals, thumbs up/down)
- Topsite tiles (sponsored and organic)
- Widgets (weather, lists, timer)
- Wallpaper and customization features
- Topic selection and section interactions

## Key Metrics & Dimensions

### User & Session Identifiers
- `client_id`: Unique client identifier
- `newtab_visit_id`: Unique visit session identifier
- `submission_date`: Date ping was received

### Configuration Settings
- Content preferences (organic/sponsored content and topsites enabled)
- Search settings (default engines for normal and private browsing)
- Widget settings (weather, topsite rows, blocked sponsors)
- UI preferences (homepage/newtab category, default UI usage)

### Interaction Metrics
All interaction metrics are computed only for visits with default UI (`is_default_ui = TRUE`):

**Search Metrics:**
- Search issued, ad clicks, ad impressions

**Content Metrics (Pocket stories):**
- Clicks, impressions, dismissals
- Thumbs up/down votes
- Separate tracking for organic vs sponsored content

**Topsite Metrics:**
- Clicks, impressions, dismissals
- Separate tracking for organic vs sponsored topsites

**Widget Metrics:**
- Weather, lists, and timer interactions
- Widget impressions

**Other Metrics:**
- Wallpaper interactions
- Topic selection and section management
- Visit duration and window dimensions

### Experiment Tracking
- `experiments`: Array of active experiments with branch assignments and enrollment IDs

## Usage Notes

1. **Partition Filter Required:** Always include `submission_date` in WHERE clause
2. **Default UI Filtering:** Most boolean metrics indicate events only in default UI
3. **Visit-Level Aggregation:** Counts represent totals within a single newtab visit
4. **Sponsored vs Organic:** Many metrics distinguish between sponsored and organic content

## Example Queries

### Daily Active Users with Newtab Opens
```sql
SELECT 
  submission_date,
  country,
  COUNT(DISTINCT client_id) AS dau
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
  AND is_newtab_opened = TRUE
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC;
```

### Sponsored Content Engagement Rate
```sql
SELECT 
  submission_date,
  channel,
  COUNTIF(sponsored_content_impression_count > 0) AS visits_with_impressions,
  COUNTIF(sponsored_content_click_count > 0) AS visits_with_clicks,
  SAFE_DIVIDE(
    COUNTIF(sponsored_content_click_count > 0),
    COUNTIF(sponsored_content_impression_count > 0)
  ) AS click_through_rate
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`
WHERE submission_date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
  AND is_default_ui = TRUE
GROUP BY 1, 2;
```

### Widget Adoption and Interaction
```sql
SELECT 
  submission_date,
  COUNTIF(newtab_weather_enabled) AS weather_enabled_visits,
  COUNTIF(widget_interaction_count > 0) AS visits_with_widget_interactions,
  SUM(widget_interaction_count) AS total_widget_interactions,
  SUM(widget_impression_count) AS total_widget_impressions
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND is_default_ui = TRUE
GROUP BY 1
ORDER BY 1 DESC;
```

### Average Visit Duration by Configuration
```sql
SELECT 
  newtab_category,
  sponsored_content_enabled,
  COUNT(*) AS visit_count,
  AVG(newtab_visit_duration) / 1000 AS avg_duration_seconds,
  APPROX_QUANTILES(newtab_visit_duration / 1000, 100)[OFFSET(50)] AS median_duration_seconds
FROM `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_visits_daily_v2`
WHERE submission_date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
  AND newtab_visit_duration IS NOT NULL
  AND newtab_visit_duration > 0
GROUP BY 1, 2
ORDER BY 3 DESC;
```

## Downstream Analysis Suggestions

### 1. Content Performance Analysis
Analyze the effectiveness of sponsored vs organic content by comparing click-through rates, dismissal rates, and sentiment (thumbs up/down) across different user segments (channel, country, configuration). Identify which content types drive the most engagement and which are frequently dismissed.

### 2. User Segmentation & Personalization
Segment users based on their newtab configuration preferences (`homepage_category`, `newtab_category`, enabled features) and interaction patterns. Analyze how different segments engage with various newtab features to inform personalization strategies and default settings.

### 3. A/B Testing & Experiment Impact
Leverage the `experiments` array to analyze the impact of feature rollouts and A/B tests on key metrics like visit duration, interaction counts, search behavior, and content engagement. Compare control vs treatment branches across multiple dimensions.

### 4. Widget & Feature Adoption Funnel
Track the adoption and retention of newer features like the weather widget, wallpaper customization, and topic selection. Analyze the relationship between feature enablement, interaction frequency, and overall newtab engagement to prioritize product development.

## Related Tables

- `firefox_desktop_stable.newtab_v1`: Source ping table with raw event data
- `firefox_desktop_derived.clients_daily_v6`: Client-level daily metrics for broader Firefox usage context

## Change Log

- **v2 (August 2024):** Added Phase 4 fields including `sample_id`, `profile_group_id`, `geo_subdivision`, `experiments`, widget metrics, search engine settings, dismissal counts, detailed interaction counts, visit duration, and window dimensions.
- **v1:** Initial version with core newtab interaction metrics
