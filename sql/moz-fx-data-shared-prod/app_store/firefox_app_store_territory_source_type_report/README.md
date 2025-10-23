# Firefox App Store Territory Source Type Report

## Overview

This view provides Apple App Store engagement metrics for Firefox iOS app, segmented by territory (country/region) and traffic source type. The data tracks how users discover and interact with Firefox in the App Store, including impressions and page views from different acquisition channels.

**Data Source**: Apple App Store Connect via Fivetran
**Timezone**: Pacific Time (PT) - Data represents full days from 12:00 AM to 11:59 PM PT
**Update Frequency**: Daily
**Reference**: [Apple App Store Connect Documentation](https://developer.apple.com/help/app-store-connect/view-sales-and-trends/download-and-view-reports)

## Schema

The table contains the following key dimensions and metrics:

### Dimensions
- **app_id**: Apple's unique identifier for the Firefox application
- **date**: Report date (stored as timestamp at midnight PT)
- **source_type**: Traffic source category (e.g., "App Referrer", "Web Referrer", "Unavailable", "Institutional Purchase")
- **territory**: Country or region name

### Metrics
- **impressions**: Total number of times Firefox was viewed in the App Store
- **impressions_unique_device**: Number of unique devices that generated impressions
- **page_views**: Total views of Firefox product page
- **page_views_unique_device**: Number of unique devices that viewed the product page

### Metadata
- **meets_threshold**: Boolean indicating whether metrics meet Apple's privacy threshold
- **_fivetran_synced**: ETL sync timestamp from Fivetran

## Important Notes

### Timezone Handling
The date field always shows midnight in the timestamp. No timezone conversion is applied in the view to avoid incorrectly shifting data by one day. All dates should be interpreted as Pacific Time.

### Privacy Threshold
Apple applies privacy thresholds to protect user privacy. When `meets_threshold` is false, the metrics may be suppressed or aggregated differently.

## Downstream Analysis Suggestions

### 1. Geographic Performance Analysis
Identify which territories drive the most App Store engagement for Firefox:
```sql
SELECT
  territory,
  SUM(impressions) as total_impressions,
  SUM(page_views) as total_page_views,
  SAFE_DIVIDE(SUM(page_views), SUM(impressions)) as impression_to_pageview_rate
FROM `moz-fx-data-shared-prod.app_store.firefox_app_store_territory_source_type_report`
WHERE meets_threshold = TRUE
  AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY territory
ORDER BY total_impressions DESC
LIMIT 20;
```

### 2. Source Type Effectiveness
Compare which traffic sources are most effective at converting impressions to page views:
```sql
SELECT
  source_type,
  SUM(impressions) as total_impressions,
  SUM(page_views) as total_page_views,
  SAFE_DIVIDE(SUM(page_views), SUM(impressions)) * 100 as conversion_rate_pct
FROM `moz-fx-data-shared-prod.app_store.firefox_app_store_territory_source_type_report`
WHERE meets_threshold = TRUE
  AND source_type != 'Institutional Purchase'
  AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY source_type
ORDER BY conversion_rate_pct DESC;
```

### 3. Trend Analysis by Territory and Source
Track how engagement metrics evolve over time for key markets:
```sql
SELECT
  DATE_TRUNC(date, WEEK) as week,
  territory,
  source_type,
  SUM(impressions_unique_device) as unique_impressions,
  SUM(page_views_unique_device) as unique_pageviews
FROM `moz-fx-data-shared-prod.app_store.firefox_app_store_territory_source_type_report`
WHERE meets_threshold = TRUE
  AND territory IN ('United States', 'Germany', 'France', 'United Kingdom')
  AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
GROUP BY week, territory, source_type
ORDER BY week DESC, territory, source_type;
```

### 4. Cross-Funnel Analysis with Downloads
Combine with download metrics to analyze the complete acquisition funnel:
```sql
SELECT
  a.territory,
  a.source_type,
  SUM(a.page_views_unique_device) as unique_pageviews,
  SUM(d.first_time_downloads) as first_time_downloads,
  SAFE_DIVIDE(SUM(d.first_time_downloads), SUM(a.page_views_unique_device)) as pageview_to_download_rate
FROM `moz-fx-data-shared-prod.app_store.firefox_app_store_territory_source_type_report` a
LEFT JOIN `moz-fx-data-shared-prod.app_store.firefox_downloads_territory_source_type_report` d
  ON a.date = d.date
  AND a.territory = d.territory
  AND a.source_type = d.source_type
WHERE a.meets_threshold = TRUE
  AND a.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY a.territory, a.source_type
HAVING unique_pageviews > 100
ORDER BY pageview_to_download_rate DESC;
```

## Related Tables

- **firefox_downloads_territory_source_type_report**: Download metrics by territory and source
- **firefox_usage_territory_source_type_report**: Usage metrics including active devices and sessions
- **firefox_app_store_territory_web_referrer_report**: App Store metrics by web referrer
- **firefox_downloads_territory_web_referrer_report**: Download metrics by web referrer

## Data Quality Considerations

- Records where `meets_threshold = FALSE` may have metrics suppressed for privacy
- The `source_type` value "Unavailable" indicates Apple could not determine the source
- "Institutional Purchase" source type typically represents volume purchase programs
- Unique device counts may be approximate due to Apple's privacy-preserving methods
