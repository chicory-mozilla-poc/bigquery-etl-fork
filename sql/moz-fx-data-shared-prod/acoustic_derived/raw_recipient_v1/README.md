# Raw Recipient - Acoustic Campaign Data

## Table Overview

The `raw_recipient_v1` table contains raw email recipient interaction data exported from Acoustic Campaign. This table tracks detailed email performance metrics including opens, clicks, bounces, and other engagement events for contacts in email marketing campaigns. The data is imported from CSV files exported by Acoustic Campaign's API and provides a comprehensive view of email recipient behavior.

**Data Source:** Acoustic Campaign API ([Raw Recipient Data Export](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport))

**Update Frequency:** Incremental daily updates

**Time Partitioning:** Daily partitions on `submission_date` field (775-day retention)

**Clustering:** Optimized for queries filtering on `event_type`, `recipient_type`, and `body_type`

## Key Columns

- **recipient_id**: Unique identifier for each contact in Acoustic Campaign
- **mailing_id**: Links events to specific sent email messages
- **campaign_id**: Associates events with marketing campaigns
- **event_type**: Categorizes the type of interaction (open, click, bounce, suppression, etc.)
- **event_timestamp**: When the email event occurred in the user's timezone
- **url**: Captured for click events to track which links were engaged
- **suppression_reason**: Explains why a contact was excluded from the mailing

## Downstream Analysis Suggestions

### 1. Email Campaign Performance Analysis
Analyze email engagement rates across campaigns to identify high-performing content and optimal send strategies.

```sql
SELECT
  campaign_id,
  event_type,
  COUNT(DISTINCT recipient_id) AS unique_recipients,
  COUNT(*) AS total_events,
  COUNT(DISTINCT mailing_id) AS mailings_sent
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY campaign_id, event_type
ORDER BY campaign_id, total_events DESC
```

### 2. Recipient Engagement Patterns
Identify recipient segments based on engagement behavior (active clickers, openers only, non-responders) to optimize targeting.

```sql
SELECT
  recipient_id,
  COUNT(DISTINCT CASE WHEN event_type = 'click' THEN mailing_id END) AS click_count,
  COUNT(DISTINCT CASE WHEN event_type = 'open' THEN mailing_id END) AS open_count,
  COUNT(DISTINCT mailing_id) AS total_mailings_received,
  MAX(event_timestamp) AS last_engagement_date
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY recipient_id
HAVING click_count > 0 OR open_count > 0
ORDER BY click_count DESC, open_count DESC
```

### 3. Link Performance and Click-Through Analysis
Track which links and content types drive the most engagement to inform content strategy.

```sql
SELECT
  click_name,
  url,
  body_type,
  COUNT(DISTINCT recipient_id) AS unique_clickers,
  COUNT(*) AS total_clicks,
  COUNT(DISTINCT mailing_id) AS mailings_with_link
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE event_type = 'click'
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 60 DAY)
  AND url IS NOT NULL
GROUP BY click_name, url, body_type
ORDER BY unique_clickers DESC
LIMIT 50
```

### 4. Suppression and Deliverability Monitoring
Monitor suppression reasons and patterns to improve list health and email deliverability rates.

```sql
SELECT
  suppression_reason,
  recipient_type,
  DATE(event_timestamp) AS suppression_date,
  COUNT(DISTINCT recipient_id) AS suppressed_recipients,
  COUNT(*) AS suppression_events
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE suppression_reason IS NOT NULL
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY suppression_reason, recipient_type, suppression_date
ORDER BY suppression_date DESC, suppressed_recipients DESC
```

## Data Quality Notes

- All columns are nullable to accommodate varying event types
- Not all fields are populated for every event type (e.g., `url` and `click_name` only for click events)
- `event_timestamp` reflects the user's timezone, not UTC
- The table uses incremental loading, so historical data builds over time
- Client-level data classification indicates PII considerations

## Related Tables

- Acoustic Campaign source data via API exports
- Email marketing campaign performance dashboards
- Recipient list management tables
