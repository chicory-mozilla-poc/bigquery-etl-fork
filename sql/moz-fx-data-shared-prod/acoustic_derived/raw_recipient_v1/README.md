# Raw Recipient (Acoustic Data)

## Overview

This table contains raw recipient-level email event data imported from Acoustic Campaign (formerly IBM Watson Campaign Automation). It captures comprehensive metrics around email performance including sends, opens, clicks, bounces, and suppressions for all email campaigns managed through the Acoustic platform.

The data is sourced directly from Acoustic Campaign's Raw Recipient Data Export API and provides granular, event-level tracking for email marketing analytics.

## Data Source

- **Platform**: Acoustic Campaign
- **Import Type**: CSV data export
- **API Reference**: [Acoustic Campaign Raw Recipient Data Export](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)
- **Update Frequency**: Daily incremental loads
- **Partitioning**: Day-partitioned by `submission_date`

## Schema

| Column Name | Type | Description |
|-------------|------|-------------|
| `recipient_id` | INTEGER | Unique identifier for the contact/recipient in Acoustic Campaign system who received or interacted with the email |
| `report_id` | INTEGER | Unique identifier for the report associated with this email event in Acoustic Campaign |
| `mailing_id` | INTEGER | Unique identifier for the sent email mailing in Acoustic Campaign |
| `campaign_id` | INTEGER | Unique identifier for the email campaign in Acoustic Campaign |
| `content_id` | STRING | Unique identifier for the email content template or creative used in the mailing |
| `event_timestamp` | DATETIME | Timestamp when the email event occurred, recorded in the API user's timezone |
| `event_type` | STRING | Type of email event (e.g., sent, delivered, opened, clicked, bounced, unsubscribed, suppressed) |
| `recipient_type` | STRING | Classification of the recipient (e.g., normal, seed, test) indicating their role in the email campaign |
| `body_type` | STRING | Email body format received by the contact (e.g., HTML, plain text, multipart) |
| `click_name` | STRING | User-defined name assigned to the clicked link or clickstream for tracking purposes |
| `url` | STRING | Full URL of the link clicked by the recipient, capturing clickthrough destinations |
| `suppression_reason` | STRING | Reason why the email was suppressed or recipient was excluded (e.g., opt-out, bounce, complaint, unsubscribe) |
| `submission_date` | DATE | Partition date for ETL processing, aligned with the date component of event_timestamp for data organization |

## Table Properties

- **Clustering**: event_type, recipient_type, body_type
- **Partition Expiration**: 775 days
- **Partition Filter**: Not required (but recommended for performance)
- **Table Type**: Client-level data
- **Incremental**: Yes

## Suggested Downstream Analyses

### 1. Email Campaign Performance Dashboard
Analyze email engagement metrics by campaign to identify high-performing campaigns and optimize future sends.

```sql
SELECT
  campaign_id,
  DATE(event_timestamp) AS event_date,
  COUNTIF(event_type = 'sent') AS total_sent,
  COUNTIF(event_type = 'delivered') AS total_delivered,
  COUNTIF(event_type = 'opened') AS total_opens,
  COUNTIF(event_type = 'clicked') AS total_clicks,
  COUNTIF(event_type = 'bounced') AS total_bounces,
  SAFE_DIVIDE(COUNTIF(event_type = 'opened'), COUNTIF(event_type = 'delivered')) AS open_rate,
  SAFE_DIVIDE(COUNTIF(event_type = 'clicked'), COUNTIF(event_type = 'delivered')) AS click_rate
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY campaign_id, event_date
ORDER BY event_date DESC, total_sent DESC;
```

### 2. Link Click Analysis
Track which links and content drive the most engagement to inform content strategy.

```sql
SELECT
  campaign_id,
  mailing_id,
  click_name,
  url,
  COUNT(DISTINCT recipient_id) AS unique_clickers,
  COUNT(*) AS total_clicks,
  DATE(MIN(event_timestamp)) AS first_click_date,
  DATE(MAX(event_timestamp)) AS last_click_date
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE event_type = 'clicked'
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY campaign_id, mailing_id, click_name, url
HAVING unique_clickers > 10
ORDER BY unique_clickers DESC
LIMIT 100;
```

### 3. Email Suppression and Deliverability Analysis
Monitor email health by tracking suppression reasons and bounce patterns to maintain list hygiene.

```sql
SELECT
  suppression_reason,
  event_type,
  recipient_type,
  COUNT(DISTINCT recipient_id) AS affected_recipients,
  COUNT(*) AS total_events,
  DATE(MIN(event_timestamp)) AS first_occurrence,
  DATE(MAX(event_timestamp)) AS last_occurrence
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE suppression_reason IS NOT NULL
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 60 DAY)
GROUP BY suppression_reason, event_type, recipient_type
ORDER BY affected_recipients DESC;
```

### 4. Recipient Engagement Segmentation
Identify highly engaged vs. inactive recipients for targeted re-engagement campaigns.

```sql
WITH recipient_activity AS (
  SELECT
    recipient_id,
    COUNT(DISTINCT mailing_id) AS mailings_received,
    COUNTIF(event_type = 'opened') AS opens,
    COUNTIF(event_type = 'clicked') AS clicks,
    MAX(event_timestamp) AS last_activity_date,
    MIN(event_timestamp) AS first_activity_date
  FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
  WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
    AND recipient_type = 'normal'
  GROUP BY recipient_id
)
SELECT
  CASE
    WHEN clicks >= 5 THEN 'Highly Engaged'
    WHEN opens >= 10 THEN 'Moderately Engaged'
    WHEN opens >= 1 THEN 'Low Engagement'
    ELSE 'No Engagement'
  END AS engagement_segment,
  COUNT(DISTINCT recipient_id) AS recipient_count,
  AVG(opens) AS avg_opens,
  AVG(clicks) AS avg_clicks,
  AVG(mailings_received) AS avg_mailings_received
FROM recipient_activity
GROUP BY engagement_segment
ORDER BY recipient_count DESC;
```

## Owners

- cbeck@mozilla.com

## Related Tables

- Consider joining with campaign metadata tables for campaign names and descriptions
- Link with content tables for detailed email creative information
- Combine with user profile data for demographic analysis

## Notes

- Events are recorded in the API user's timezone
- The table includes both normal recipients and test/seed recipients (filterable via `recipient_type`)
- Click events include both `click_name` and `url` for comprehensive link tracking
- Data retention is 775 days based on partition expiration settings
