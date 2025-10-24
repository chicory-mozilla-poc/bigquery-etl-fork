# Raw Recipient (Acoustic Data)

## Overview

This table contains raw recipient-level email event data imported from Acoustic Campaign via CSV exports. It captures comprehensive metrics around email performance including sends, opens, clicks, bounces, and suppressions. The data provides detailed tracking of individual contact interactions with marketing emails.

**Source:** Acoustic Campaign Raw Recipient Data Export API
**Reference:** https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport

## Table Details

- **Dataset:** `moz-fx-data-shared-prod.acoustic_derived`
- **Table:** `raw_recipient_v1`
- **Type:** Client-level, incremental raw data import
- **Partitioning:** Daily partitioning on `submission_date` (775 days retention)
- **Clustering:** Optimized by `event_type`, `recipient_type`, `body_type`

## Key Metrics

This table tracks the following email event types:
- **Sent events:** Email successfully delivered to recipient
- **Open events:** Recipient opened the email
- **Click events:** Recipient clicked links within the email
- **Bounce events:** Email delivery failures
- **Suppression events:** Recipients excluded from mailings

## Schema Summary

The table contains 13 columns including:
- **Identifiers:** recipient_id, mailing_id, campaign_id, content_id
- **Event details:** event_timestamp, event_type, recipient_type
- **Engagement data:** click_name, url, body_type
- **Metadata:** report_id, suppression_reason, submission_date

## Downstream Analysis Suggestions

### 1. Email Engagement Rate Analysis
Calculate open rates, click-through rates (CTR), and click-to-open rates (CTOR) by campaign, content, or time period:
```sql
SELECT
  campaign_id,
  DATE(event_timestamp) as event_date,
  COUNTIF(event_type = 'Sent') as total_sent,
  COUNTIF(event_type = 'Open') as total_opens,
  COUNTIF(event_type = 'Click') as total_clicks,
  SAFE_DIVIDE(COUNTIF(event_type = 'Open'), COUNTIF(event_type = 'Sent')) as open_rate,
  SAFE_DIVIDE(COUNTIF(event_type = 'Click'), COUNTIF(event_type = 'Sent')) as ctr
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
GROUP BY campaign_id, event_date
```

### 2. Link Performance Analysis
Identify top-performing links and CTAs by analyzing click patterns:
```sql
SELECT
  click_name,
  url,
  COUNT(DISTINCT recipient_id) as unique_clickers,
  COUNT(*) as total_clicks
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE event_type = 'Click' AND click_name IS NOT NULL
GROUP BY click_name, url
ORDER BY unique_clickers DESC
```

### 3. Email Deliverability & Suppression Analysis
Monitor email health by tracking bounce rates and suppression reasons:
```sql
SELECT
  DATE(event_timestamp) as event_date,
  suppression_reason,
  recipient_type,
  COUNT(*) as suppression_count
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE suppression_reason IS NOT NULL
GROUP BY event_date, suppression_reason, recipient_type
ORDER BY event_date DESC, suppression_count DESC
```

### 4. Body Type Performance Comparison
Compare engagement metrics across HTML vs plain text email formats:
```sql
SELECT
  body_type,
  COUNTIF(event_type = 'Sent') as sent,
  COUNTIF(event_type = 'Open') as opens,
  COUNTIF(event_type = 'Click') as clicks,
  SAFE_DIVIDE(COUNTIF(event_type = 'Open'), COUNTIF(event_type = 'Sent')) as open_rate,
  SAFE_DIVIDE(COUNTIF(event_type = 'Click'), COUNTIF(event_type = 'Open')) as ctor
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
GROUP BY body_type
```

## Owners

- **Primary Contact:** cbeck@mozilla.com

## Notes

- Data is incrementally loaded via Airflow ETL processes
- The `submission_date` field should align with the date component of `event_timestamp`
- Partition filtering can improve query performance for date-range analyses
- Clustering on event_type, recipient_type, and body_type optimizes common filter patterns
