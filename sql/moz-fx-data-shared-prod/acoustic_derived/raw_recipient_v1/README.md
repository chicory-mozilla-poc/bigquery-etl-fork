# Raw Recipient (Acoustic Campaign Data)

## Overview

This table contains raw recipient-level email event data exported from Acoustic Campaign (formerly Silverpop). It captures detailed metrics about email campaign performance, including sends, opens, clicks, bounces, and suppressions. The data is imported from CSV files exported via the Acoustic Campaign API.

## Data Source

- **Platform**: Acoustic Campaign
- **Import Method**: CSV data export
- **API Reference**: [Acoustic Campaign Raw Recipient Data Export](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)

## Table Structure

### Partitioning
- **Field**: `submission_date`
- **Type**: Daily partitioning
- **Retention**: 775 days (~2 years)

### Clustering
Optimized for queries filtering on:
- `event_type` - Email event categories
- `recipient_type` - Contact classifications
- `body_type` - Email format types

## Key Columns

### Identifiers
- **recipient_id**: Unique contact identifier in Acoustic
- **mailing_id**: Specific email send identifier
- **campaign_id**: Marketing campaign identifier
- **content_id**: Email template/content identifier
- **report_id**: Associated report identifier

### Event Data
- **event_type**: Type of interaction (sent, opened, clicked, bounced, etc.)
- **event_timestamp**: When the event occurred (API user's timezone)
- **recipient_type**: Contact classification in Acoustic
- **body_type**: Email format (HTML, plain text)

### Engagement Details
- **click_name**: Named link or clickstream label
- **url**: Full clicked URL
- **suppression_reason**: Why contact was excluded from receiving email

### ETL Metadata
- **submission_date**: Server-side ingestion date (aligns with event_timestamp)

## Downstream Analysis Suggestions

### 1. Email Campaign Performance Analysis
Analyze open rates, click-through rates, and engagement metrics by campaign:
```sql
SELECT
  campaign_id,
  mailing_id,
  COUNT(DISTINCT recipient_id) as total_recipients,
  COUNTIF(event_type = 'opened') as opens,
  COUNTIF(event_type = 'clicked') as clicks,
  COUNTIF(event_type = 'bounced') as bounces,
  SAFE_DIVIDE(COUNTIF(event_type = 'opened'), COUNT(DISTINCT recipient_id)) as open_rate,
  SAFE_DIVIDE(COUNTIF(event_type = 'clicked'), COUNTIF(event_type = 'opened')) as click_to_open_rate
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY campaign_id, mailing_id
ORDER BY total_recipients DESC
```

### 2. Link Performance and Content Engagement
Identify top-performing links and content across campaigns:
```sql
SELECT
  click_name,
  url,
  body_type,
  COUNT(DISTINCT recipient_id) as unique_clickers,
  COUNT(*) as total_clicks
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE event_type = 'clicked'
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY click_name, url, body_type
ORDER BY unique_clickers DESC
LIMIT 100
```

### 3. Recipient Behavior Segmentation
Segment recipients by engagement patterns and types:
```sql
SELECT
  recipient_type,
  body_type,
  COUNT(DISTINCT recipient_id) as unique_recipients,
  COUNT(*) as total_events,
  COUNTIF(event_type IN ('opened', 'clicked')) as engaged_events,
  SAFE_DIVIDE(COUNTIF(event_type IN ('opened', 'clicked')), COUNT(*)) as engagement_rate
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 60 DAY)
GROUP BY recipient_type, body_type
ORDER BY unique_recipients DESC
```

### 4. Suppression and Deliverability Analysis
Monitor email deliverability issues and suppression patterns:
```sql
SELECT
  suppression_reason,
  event_type,
  DATE_TRUNC(submission_date, WEEK) as week,
  COUNT(DISTINCT recipient_id) as affected_recipients,
  COUNT(*) as event_count
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE suppression_reason IS NOT NULL
  OR event_type IN ('bounced', 'unsubscribed', 'suppressed')
GROUP BY suppression_reason, event_type, week
ORDER BY week DESC, event_count DESC
```

## Owner

- **Email**: gkatre@mozilla.com

## Additional Resources

- [Acoustic Campaign Documentation](https://developer.goacoustic.com/acoustic-campaign/docs)
- [Acoustic Campaign API Reference](https://developer.goacoustic.com/acoustic-campaign/reference)
