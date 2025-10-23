# Raw Recipient - Acoustic Campaign Email Events

## Overview

This table contains raw email event data exported from Acoustic Campaign (formerly IBM Watson Campaign Automation). It captures detailed metrics and events related to email campaign performance, including sends, opens, clicks, bounces, and suppressions.

**Data Source**: Acoustic Campaign Raw Recipient Data Export API
**Reference**: https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport

## Table Details

- **Dataset**: `moz-fx-data-shared-prod.acoustic_derived`
- **Table**: `raw_recipient_v1`
- **Type**: Client-level, Incremental
- **Partitioning**: Daily partition on `submission_date` (775 days retention)
- **Clustering**: `event_type`, `recipient_type`, `body_type`

## Schema Summary

The table includes 13 columns tracking:
- **Identifiers**: Contact/recipient, mailing, campaign, and content IDs
- **Event Details**: Timestamp, type, and recipient classification
- **Email Properties**: Body type, click tracking, URLs
- **Delivery Status**: Suppression reasons for blocked emails
- **ETL Metadata**: Submission date for data pipeline tracking

## Key Use Cases

### 1. Email Campaign Performance Analysis
```sql
SELECT
  campaign_id,
  event_type,
  COUNT(DISTINCT recipient_id) AS unique_recipients,
  COUNT(*) AS total_events
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY campaign_id, event_type
ORDER BY campaign_id, total_events DESC;
```
Track engagement metrics (opens, clicks) across different email campaigns to measure effectiveness.

### 2. Recipient Engagement Funnel
```sql
SELECT
  DATE(event_timestamp) AS event_date,
  COUNTIF(event_type = 'sent') AS emails_sent,
  COUNTIF(event_type = 'opened') AS emails_opened,
  COUNTIF(event_type = 'clicked') AS emails_clicked,
  SAFE_DIVIDE(COUNTIF(event_type = 'opened'), COUNTIF(event_type = 'sent')) AS open_rate,
  SAFE_DIVIDE(COUNTIF(event_type = 'clicked'), COUNTIF(event_type = 'opened')) AS click_through_rate
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY event_date
ORDER BY event_date DESC;
```
Analyze email engagement funnel metrics over time to identify trends in recipient behavior.

### 3. Link Click Analysis
```sql
SELECT
  click_name,
  url,
  COUNT(DISTINCT recipient_id) AS unique_clickers,
  COUNT(*) AS total_clicks
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE event_type = 'clicked'
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
  AND url IS NOT NULL
GROUP BY click_name, url
ORDER BY total_clicks DESC
LIMIT 20;
```
Identify the most clicked links and content to understand what drives recipient engagement.

### 4. Email Deliverability and Suppression Analysis
```sql
SELECT
  suppression_reason,
  recipient_type,
  body_type,
  COUNT(DISTINCT recipient_id) AS affected_recipients,
  COUNT(*) AS suppression_events
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE suppression_reason IS NOT NULL
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 60 DAY)
GROUP BY suppression_reason, recipient_type, body_type
ORDER BY suppression_events DESC;
```
Monitor email deliverability issues and understand why emails are being suppressed or blocked.

## Data Refresh

This table is incrementally updated via Airflow ETL pipelines. The `submission_date` field represents the Airflow execution date and should align with the dates in the `event_timestamp` field.

## Owner

- **Primary Contact**: gkatre@mozilla.com

## Related Tables

For additional Acoustic Campaign data, explore other tables in the `acoustic_derived` dataset.
