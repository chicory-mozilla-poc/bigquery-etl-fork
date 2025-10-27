# Raw Recipient (Acoustic Data)

## Table Overview

`moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1` contains raw recipient-level event data exported from Acoustic Campaign. This table captures detailed email engagement metrics including sends, opens, clicks, bounces, and other email performance indicators at the individual recipient level.

## Data Source

- **Platform**: Acoustic Campaign (formerly IBM Watson Campaign Automation)
- **Import Method**: CSV data export from Acoustic
- **API Reference**: [Acoustic Raw Recipient Data Export](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)

## Table Characteristics

- **Partitioning**: Daily partitioning on `submission_date` field (775 days retention)
- **Clustering**: Optimized for queries filtering on `event_type`, `recipient_type`, and `body_type`
- **Granularity**: Contact-level events
- **Update Frequency**: Incremental daily updates

## Key Use Cases

This table serves as the foundation for:
- Email campaign performance analysis
- Recipient engagement tracking
- Deliverability monitoring
- Campaign attribution and ROI analysis

## Schema Summary

The table contains 13 columns organized into two categories:

### Acoustic Campaign Fields (12 columns)
- **Identifiers**: `recipient_id`, `report_id`, `mailing_id`, `campaign_id`, `content_id`
- **Event Data**: `event_timestamp`, `event_type`
- **Recipient Context**: `recipient_type`, `body_type`
- **Click Tracking**: `click_name`, `url`
- **Suppression**: `suppression_reason`

### ETL Fields (1 column)
- `submission_date`: Data import processing date

## Downstream Analysis Suggestions

### 1. Email Campaign Performance Dashboard
```sql
-- Analyze campaign engagement rates over time
SELECT
  campaign_id,
  DATE(event_timestamp) as event_date,
  event_type,
  COUNT(DISTINCT recipient_id) as unique_recipients,
  COUNT(*) as total_events
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY campaign_id, event_date, event_type
ORDER BY event_date DESC, campaign_id;
```

### 2. Click-Through Rate Analysis by Content
```sql
-- Identify top-performing links and content
SELECT
  content_id,
  click_name,
  url,
  COUNT(DISTINCT recipient_id) as unique_clickers,
  COUNT(*) as total_clicks
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE event_type = 'click'
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY content_id, click_name, url
ORDER BY unique_clickers DESC
LIMIT 20;
```

### 3. Suppression and Deliverability Monitoring
```sql
-- Track suppression reasons and email health
SELECT
  suppression_reason,
  body_type,
  DATE(event_timestamp) as event_date,
  COUNT(DISTINCT recipient_id) as suppressed_recipients
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE suppression_reason IS NOT NULL
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 60 DAY)
GROUP BY suppression_reason, body_type, event_date
ORDER BY event_date DESC, suppressed_recipients DESC;
```

### 4. Recipient Engagement Cohort Analysis
```sql
-- Build recipient engagement profiles by analyzing event sequences
SELECT
  recipient_id,
  COUNT(DISTINCT mailing_id) as mailings_received,
  COUNTIF(event_type = 'open') as opens,
  COUNTIF(event_type = 'click') as clicks,
  MAX(event_timestamp) as last_engagement
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
GROUP BY recipient_id
HAVING mailings_received >= 5
ORDER BY clicks DESC, opens DESC;
```

## Data Quality Notes

- Events are recorded in the API user's timezone
- The `submission_date` field aligns with Airflow execution date and should overlap with dates in `event_timestamp`
- Not all fields are populated for every event type (e.g., `url` and `click_name` are only relevant for click events)

## Owner

- cbeck@mozilla.com

## Related Tables

- `moz-fx-data-shared-prod.acoustic_external.raw_recipient_raw_v1` - Raw import source
- `moz-fx-data-shared-prod.acoustic_derived.contact_v1` - Contact master data
