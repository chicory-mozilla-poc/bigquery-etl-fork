# Raw Recipient (Acoustic Campaign Data)

## Overview

This table contains raw recipient-level event data exported from Acoustic Campaign via CSV. It captures detailed email performance metrics including sends, opens, clicks, bounces, and suppressions for email campaigns managed through the Acoustic platform.

**Source:** [Acoustic Campaign API - Raw Recipient Data Export](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)

## Table Details

- **Database:** `moz-fx-data-shared-prod`
- **Dataset:** `acoustic_derived`
- **Table:** `raw_recipient_v1`
- **Update Frequency:** Incremental daily loads
- **Partitioning:** Day-partitioned by `submission_date` (775-day expiration)
- **Clustering:** `event_type`, `recipient_type`, `body_type`

## Schema Summary

The table contains 14 columns tracking:
- **Recipient Identifiers:** `recipient_id`, `recipient_type`
- **Campaign Identifiers:** `campaign_id`, `mailing_id`, `report_id`, `content_id`
- **Event Details:** `event_type`, `event_timestamp`, `body_type`
- **Interaction Data:** `click_name`, `url`
- **Suppression Info:** `suppression_reason`
- **ETL Metadata:** `submission_date`

## Downstream Analysis Suggestions

### 1. Email Engagement Analysis
Calculate open rates, click-through rates (CTR), and conversion metrics by campaign, content type, or time period:
```sql
SELECT
  campaign_id,
  DATE(event_timestamp) as event_date,
  COUNTIF(event_type = 'Sent') as emails_sent,
  COUNTIF(event_type = 'Open') as opens,
  COUNTIF(event_type = 'Click') as clicks,
  SAFE_DIVIDE(COUNTIF(event_type = 'Open'), COUNTIF(event_type = 'Sent')) as open_rate,
  SAFE_DIVIDE(COUNTIF(event_type = 'Click'), COUNTIF(event_type = 'Sent')) as ctr
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
GROUP BY campaign_id, event_date
```

### 2. Link Performance Tracking
Identify top-performing links and content by analyzing click patterns across campaigns:
```sql
SELECT
  click_name,
  url,
  COUNT(DISTINCT recipient_id) as unique_clickers,
  COUNT(*) as total_clicks,
  COUNT(DISTINCT campaign_id) as campaigns_used
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE event_type = 'Click'
GROUP BY click_name, url
ORDER BY unique_clickers DESC
```

### 3. Suppression and Deliverability Analysis
Monitor email deliverability issues and understand suppression patterns to improve list hygiene:
```sql
SELECT
  suppression_reason,
  event_type,
  COUNT(*) as event_count,
  COUNT(DISTINCT recipient_id) as affected_recipients
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE suppression_reason IS NOT NULL
GROUP BY suppression_reason, event_type
ORDER BY event_count DESC
```

### 4. Recipient Behavior Cohort Analysis
Segment recipients by engagement patterns to identify power users vs. inactive subscribers:
```sql
SELECT
  recipient_id,
  COUNT(DISTINCT CASE WHEN event_type = 'Open' THEN mailing_id END) as emails_opened,
  COUNT(DISTINCT CASE WHEN event_type = 'Click' THEN mailing_id END) as emails_clicked,
  MIN(event_timestamp) as first_interaction,
  MAX(event_timestamp) as last_interaction
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE event_type IN ('Open', 'Click')
GROUP BY recipient_id
```

## Owners

- cbeck@mozilla.com

## Notes

- This table contains raw, unprocessed event data from Acoustic Campaign
- Events are clustered by type for efficient querying of specific interaction patterns
- Use `submission_date` for date-based filtering to leverage partitioning
- Partition filter is not required but recommended for performance
