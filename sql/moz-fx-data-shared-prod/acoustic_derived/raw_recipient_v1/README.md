# Raw Recipient (Acoustic Campaign Data)

## Overview

This table contains raw event-level data exported from Acoustic Campaign's Raw Recipient Data Export API. It captures granular email engagement metrics including sends, opens, clicks, bounces, and suppressions. The data is imported from CSV files provided by Acoustic and serves as the foundation for email performance analysis and reporting.

## Key Characteristics

- **Data Source**: Acoustic Campaign Raw Recipient Data Export API
- **Granularity**: Event-level (one row per recipient event)
- **Time Partitioning**: Daily partitions on `submission_date` (775 days retention)
- **Clustering**: Optimized for queries filtering by `event_type`, `recipient_type`, and `body_type`
- **Update Frequency**: Incremental daily loads via Airflow

## Schema Summary

The table includes:
- **Identifiers**: recipient_id, mailing_id, campaign_id, content_id
- **Event Details**: event_type, event_timestamp, recipient_type
- **Engagement Data**: click_name, url (for click events)
- **Email Format**: body_type
- **Suppression Tracking**: suppression_reason
- **ETL Metadata**: submission_date

## Suggested Downstream Analyses

### 1. Email Engagement Funnel Analysis
Analyze the progression of recipients through the email engagement funnel (sent → opened → clicked) by campaign and content variant. Calculate conversion rates at each stage to identify high-performing campaigns and content.

```sql
SELECT
  campaign_id,
  content_id,
  COUNTIF(event_type = 'sent') AS sends,
  COUNTIF(event_type = 'opened') AS opens,
  COUNTIF(event_type = 'clicked') AS clicks,
  SAFE_DIVIDE(COUNTIF(event_type = 'opened'), COUNTIF(event_type = 'sent')) AS open_rate,
  SAFE_DIVIDE(COUNTIF(event_type = 'clicked'), COUNTIF(event_type = 'opened')) AS click_through_rate
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY campaign_id, content_id
ORDER BY sends DESC
```

### 2. Suppression Trend Analysis
Monitor suppression patterns over time to identify deliverability issues, content problems, or audience fatigue. Track suppression reasons by campaign to improve targeting and content strategies.

```sql
SELECT
  submission_date,
  suppression_reason,
  COUNT(DISTINCT recipient_id) AS suppressed_recipients,
  COUNT(*) AS suppression_events
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE suppression_reason IS NOT NULL
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY submission_date, suppression_reason
ORDER BY submission_date DESC, suppressed_recipients DESC
```

### 3. Click-Level Performance by Link
Identify which links and CTAs drive the most engagement by analyzing click patterns. Use this to optimize email content placement and messaging.

```sql
SELECT
  click_name,
  url,
  COUNT(DISTINCT recipient_id) AS unique_clickers,
  COUNT(*) AS total_clicks,
  COUNT(DISTINCT mailing_id) AS mailings_used_in
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE event_type = 'clicked'
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY click_name, url
ORDER BY unique_clickers DESC
LIMIT 50
```

### 4. Body Type Effectiveness Comparison
Compare engagement rates across different email body types (HTML vs plain text) to determine which format resonates better with recipients.

```sql
SELECT
  body_type,
  recipient_type,
  COUNT(DISTINCT CASE WHEN event_type = 'sent' THEN recipient_id END) AS recipients_sent,
  COUNT(DISTINCT CASE WHEN event_type = 'opened' THEN recipient_id END) AS recipients_opened,
  COUNT(DISTINCT CASE WHEN event_type = 'clicked' THEN recipient_id END) AS recipients_clicked,
  SAFE_DIVIDE(
    COUNT(DISTINCT CASE WHEN event_type = 'opened' THEN recipient_id END),
    COUNT(DISTINCT CASE WHEN event_type = 'sent' THEN recipient_id END)
  ) AS unique_open_rate,
  SAFE_DIVIDE(
    COUNT(DISTINCT CASE WHEN event_type = 'clicked' THEN recipient_id END),
    COUNT(DISTINCT CASE WHEN event_type = 'sent' THEN recipient_id END)
  ) AS unique_click_rate
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY body_type, recipient_type
ORDER BY recipients_sent DESC
```

## Data Quality Notes

- The `report_id` field may be null for certain event types
- `click_name` and `url` are only populated for click events
- `suppression_reason` is only populated when a contact is suppressed
- Event timestamps are in the API user's time zone, not UTC

## References

- [Acoustic Campaign Raw Recipient Data Export API Documentation](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)
