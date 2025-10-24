# Raw Recipient (Acoustic Campaign Data)

## Overview

This table contains raw recipient-level event data exported from Acoustic Campaign, a marketing automation platform. It captures detailed metrics around email campaign performance including sends, opens, clicks, bounces, and suppressions. The data is imported from CSV exports provided by Acoustic Campaign's API.

## Data Source

- **Platform**: Acoustic Campaign (formerly Silverpop)
- **API Documentation**: [Raw Recipient Data Export](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)
- **Import Type**: CSV file export
- **Update Frequency**: Incremental (daily)

## Table Details

- **Project**: moz-fx-data-shared-prod
- **Dataset**: acoustic_derived
- **Table**: raw_recipient_v1
- **Level**: Client-level event data
- **Partitioning**: Daily partition by `submission_date` (775 days retention)
- **Clustering**: `event_type`, `recipient_type`, `body_type`

## Schema Summary

The table contains 13 columns tracking:
- **Identifiers**: Contact, mailing, campaign, and content IDs
- **Event details**: Timestamp, event type, and recipient classification
- **Email properties**: Body type and content format
- **Click tracking**: Link names and URLs for clickthrough events
- **Suppression tracking**: Reasons for email delivery failures or opt-outs

## Key Columns

- `recipient_id`: Unique contact identifier for joining with other recipient data
- `event_type`: Critical field for filtering specific event types (sent, opened, clicked, bounced)
- `event_timestamp`: Timestamp of the event occurrence
- `mailing_id`: Links events to specific email sends
- `campaign_id`: Groups events by marketing campaign
- `url` & `click_name`: Track which links were clicked in emails
- `suppression_reason`: Identifies why emails failed to deliver

## Downstream Analysis Suggestions

### 1. Email Campaign Performance Dashboard
Analyze email engagement metrics by campaign to measure effectiveness:
```sql
SELECT
  campaign_id,
  COUNT(DISTINCT CASE WHEN event_type = 'Sent' THEN recipient_id END) as total_sent,
  COUNT(DISTINCT CASE WHEN event_type = 'Open' THEN recipient_id END) as total_opens,
  COUNT(DISTINCT CASE WHEN event_type = 'Click' THEN recipient_id END) as total_clicks,
  SAFE_DIVIDE(
    COUNT(DISTINCT CASE WHEN event_type = 'Open' THEN recipient_id END),
    COUNT(DISTINCT CASE WHEN event_type = 'Sent' THEN recipient_id END)
  ) * 100 as open_rate,
  SAFE_DIVIDE(
    COUNT(DISTINCT CASE WHEN event_type = 'Click' THEN recipient_id END),
    COUNT(DISTINCT CASE WHEN event_type = 'Sent' THEN recipient_id END)
  ) * 100 as click_through_rate
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY campaign_id
ORDER BY total_sent DESC;
```

### 2. Link Performance Analysis
Identify which email links drive the most engagement:
```sql
SELECT
  click_name,
  url,
  COUNT(DISTINCT recipient_id) as unique_clickers,
  COUNT(*) as total_clicks,
  COUNT(DISTINCT mailing_id) as mailings_used_in
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE event_type = 'Click'
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY click_name, url
ORDER BY unique_clickers DESC
LIMIT 20;
```

### 3. Suppression and Deliverability Report
Monitor email deliverability issues and understand why emails are being suppressed:
```sql
SELECT
  suppression_reason,
  DATE_TRUNC(submission_date, WEEK) as week,
  COUNT(DISTINCT recipient_id) as affected_recipients,
  COUNT(*) as total_suppressions
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE suppression_reason IS NOT NULL
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
GROUP BY suppression_reason, week
ORDER BY week DESC, affected_recipients DESC;
```

### 4. Content Type and Body Format Analysis
Compare engagement rates across different email body types:
```sql
SELECT
  body_type,
  recipient_type,
  COUNT(DISTINCT CASE WHEN event_type = 'Sent' THEN recipient_id END) as sent,
  COUNT(DISTINCT CASE WHEN event_type = 'Open' THEN recipient_id END) as opened,
  COUNT(DISTINCT CASE WHEN event_type = 'Click' THEN recipient_id END) as clicked,
  SAFE_DIVIDE(
    COUNT(DISTINCT CASE WHEN event_type = 'Open' THEN recipient_id END),
    COUNT(DISTINCT CASE WHEN event_type = 'Sent' THEN recipient_id END)
  ) * 100 as open_rate_pct
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY body_type, recipient_type
ORDER BY sent DESC;
```

## Owners

- cbeck@mozilla.com

## Related Tables

This raw recipient data may be joined with:
- Campaign metadata tables for campaign names and details
- Contact/recipient dimension tables for user demographics
- Other Acoustic derived tables for comprehensive marketing analytics

## Notes

- Data is partitioned by `submission_date` for efficient querying
- Always filter by `submission_date` to optimize query performance
- Table uses clustering on `event_type`, `recipient_type`, and `body_type` for faster filtering
- Partition expiration is set to 775 days (~2 years)
- The `event_timestamp` field represents the actual time of the event, while `submission_date` is used for ETL organization
