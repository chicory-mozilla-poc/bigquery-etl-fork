# Raw Recipient Table (Acoustic Campaign Data)

## Overview

This table contains raw email recipient event data exported from Acoustic Campaign. It captures comprehensive metrics around email performance including sends, opens, clicks, bounces, and other email interactions.

## Data Source

The data is imported from Acoustic Campaign's Raw Recipient Data Export API. Reference: [Acoustic Campaign API Documentation](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)

## Table Structure

**Database**: `moz-fx-data-shared-prod`
**Dataset**: `acoustic_derived`
**Table**: `raw_recipient_v1`

### Partitioning & Clustering

- **Partitioning**: Time-based partitioning on `submission_date` (daily)
- **Partition Expiration**: 775 days
- **Clustering**: `event_type`, `recipient_type`, `body_type`

## Key Columns

### Identifiers
- `recipient_id`: Links events to specific contacts in Acoustic
- `mailing_id`: Identifies the specific email sent
- `campaign_id`: Groups events by campaign
- `content_id`: Tracks which content version was used

### Event Data
- `event_type`: Primary dimension for analyzing email interactions
- `event_timestamp`: Precise timing of each event
- `url` & `click_name`: Track link engagement

### Categorization
- `recipient_type`: Segment contacts by type
- `body_type`: Distinguish HTML vs text email performance
- `suppression_reason`: Understand delivery failures

## Suggested Downstream Analyses

### 1. Email Campaign Performance Dashboard
Analyze open rates, click-through rates, and conversion metrics by campaign. Group by `campaign_id` and `event_type` to calculate key email marketing KPIs.

```sql
SELECT
  campaign_id,
  COUNT(DISTINCT CASE WHEN event_type = 'sent' THEN recipient_id END) as total_sent,
  COUNT(DISTINCT CASE WHEN event_type = 'opened' THEN recipient_id END) as total_opened,
  COUNT(DISTINCT CASE WHEN event_type = 'clicked' THEN recipient_id END) as total_clicked
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY campaign_id
```

### 2. Content Performance Analysis
Compare engagement rates across different email content versions and body types to optimize future campaigns.

```sql
SELECT
  content_id,
  body_type,
  COUNT(DISTINCT recipient_id) as unique_recipients,
  COUNTIF(event_type = 'opened') / COUNTIF(event_type = 'sent') as open_rate,
  COUNTIF(event_type = 'clicked') / COUNTIF(event_type = 'opened') as click_rate
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY content_id, body_type
ORDER BY open_rate DESC
```

### 3. Link Engagement Tracking
Identify top-performing links and clickstreams to understand what content drives the most engagement.

```sql
SELECT
  click_name,
  url,
  COUNT(*) as total_clicks,
  COUNT(DISTINCT recipient_id) as unique_clickers
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE event_type = 'clicked'
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY click_name, url
ORDER BY total_clicks DESC
LIMIT 20
```

### 4. Suppression & Deliverability Analysis
Monitor email deliverability issues and understand why contacts are being suppressed.

```sql
SELECT
  suppression_reason,
  recipient_type,
  COUNT(DISTINCT recipient_id) as affected_contacts,
  COUNT(*) as total_suppressions
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE suppression_reason IS NOT NULL
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY suppression_reason, recipient_type
ORDER BY total_suppressions DESC
```

## Data Freshness

This is an incremental table with daily updates via Airflow. The `submission_date` should align with dates in the `event_timestamp` field.

## Owner

- **Contact**: gkatre@mozilla.com
