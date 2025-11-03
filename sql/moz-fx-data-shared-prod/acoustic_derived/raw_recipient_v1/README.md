# raw_recipient_v1

## Overview

The `raw_recipient_v1` table contains raw event-level data exported from Acoustic Campaign (formerly Silverpop), a marketing automation platform used by Mozilla. This table captures detailed email engagement metrics including sends, opens, clicks, bounces, and unsubscribes for email campaigns managed through Acoustic.

This is a **client-level** table that provides granular tracking of individual recipient interactions with marketing emails. The data is imported daily via CSV exports from Acoustic's API and serves as a foundational dataset for email performance analysis and marketing attribution.

## Schema

The table contains 14 fields organized into two categories:

### Event & Campaign Identifiers
- `recipient_id`: Unique identifier for the contact/recipient
- `report_id`: Report identifier (specific to Acoustic's reporting structure)
- `mailing_id`: Unique identifier for the sent email
- `campaign_id`: Campaign identifier grouping related mailings
- `content_id`: Content/creative variant identifier

### Event Details
- `event_timestamp`: Timestamp of the event in API user's timezone
- `event_type`: Type of engagement (e.g., "Sent", "Open", "Click", "Bounce", "Unsubscribe")
- `recipient_type`: Classification of the recipient (e.g., "To", "CC", "BCC")
- `body_type`: Email format received by contact (e.g., "HTML", "Text")

### Engagement Data
- `click_name`: User-defined name for clicked links or clickstreams
- `url`: Full hyperlink URL for click-through events
- `suppression_reason`: Reason code if recipient was suppressed from future sends

### ETL Metadata
- `submission_date`: Airflow execution date (partitioning key) aligned with event_timestamp dates

## Data Sources

**Primary Source:** Acoustic Campaign Platform
- **API Documentation:** [Acoustic Developer - Raw Recipient Data Export](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)
- **Export Format:** CSV files
- **Data Type:** Event-level email engagement metrics

**Related External Tables:**
- `acoustic_external.suppression_list_v1` - Email suppression data

## Update Schedule

- **Frequency:** Daily incremental loads
- **Partition Field:** `submission_date` (day-level partitioning)
- **Partition Retention:** 775 days (~2.1 years)
- **Orchestration:** Airflow DAG managed by Data Engineering team
- **Incremental:** Yes - new events appended daily

## Performance Optimization

**Partitioning:**
- Time-partitioned by `submission_date` (day granularity)
- Partition filter not required but recommended for query performance

**Clustering:**
- Clustered on: `event_type`, `recipient_type`, `body_type`
- Optimizes queries filtering or grouping by these dimensions

## Usage Examples

### Query recent email opens
```sql
SELECT
  recipient_id,
  mailing_id,
  event_timestamp,
  body_type
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
  AND event_type = 'Open'
ORDER BY event_timestamp DESC
LIMIT 100;
```

### Analyze click-through rates by campaign
```sql
SELECT
  campaign_id,
  COUNT(DISTINCT CASE WHEN event_type = 'Sent' THEN recipient_id END) AS recipients_sent,
  COUNT(DISTINCT CASE WHEN event_type = 'Click' THEN recipient_id END) AS recipients_clicked,
  SAFE_DIVIDE(
    COUNT(DISTINCT CASE WHEN event_type = 'Click' THEN recipient_id END),
    COUNT(DISTINCT CASE WHEN event_type = 'Sent' THEN recipient_id END)
  ) AS click_through_rate
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY campaign_id
ORDER BY recipients_sent DESC;
```

### Track bounce and suppression patterns
```sql
SELECT
  event_type,
  suppression_reason,
  COUNT(*) AS event_count,
  COUNT(DISTINCT recipient_id) AS unique_recipients
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  AND event_type IN ('HardBounce', 'SoftBounce', 'Suppressed')
GROUP BY event_type, suppression_reason
ORDER BY event_count DESC;
```

### Email engagement funnel analysis
```sql
WITH engagement AS (
  SELECT
    mailing_id,
    recipient_id,
    MAX(CASE WHEN event_type = 'Sent' THEN 1 ELSE 0 END) AS sent,
    MAX(CASE WHEN event_type = 'Open' THEN 1 ELSE 0 END) AS opened,
    MAX(CASE WHEN event_type = 'Click' THEN 1 ELSE 0 END) AS clicked
  FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
  WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
  GROUP BY mailing_id, recipient_id
)
SELECT
  mailing_id,
  SUM(sent) AS total_sent,
  SUM(opened) AS total_opened,
  SUM(clicked) AS total_clicked,
  SAFE_DIVIDE(SUM(opened), SUM(sent)) AS open_rate,
  SAFE_DIVIDE(SUM(clicked), SUM(sent)) AS click_rate
FROM engagement
GROUP BY mailing_id
ORDER BY total_sent DESC;
```

## Related Tables

- `acoustic_external.suppression_list_v1` - Email suppression and unsubscribe data
- `marketing_suppression_list_derived.main_suppression_list_v1` - Consolidated suppression list across platforms

## Notes

- **Event Types:** Common values include "Sent", "Open", "Click", "HardBounce", "SoftBounce", "Unsubscribe", "Suppressed"
- **Timezone Consideration:** `event_timestamp` reflects the API user's configured timezone; consider converting to UTC for cross-platform analysis
- **Deduplication:** Multiple events of the same type for a single recipient/mailing are possible (e.g., multiple opens)
- **NULL Values:** Many fields are nullable; `suppression_reason` is only populated for suppression events
- **Data Quality:** Monitor for delayed or missing data loads via `submission_date` gaps
- **Privacy:** Contains contact-level PII; follow Mozilla data handling policies

## Ownership

- **Owner:** cbeck@mozilla.com
- **Team:** Marketing Data Engineering

## References

- [Acoustic Campaign Developer Portal](https://developer.goacoustic.com/acoustic-campaign)
- [Raw Recipient Data Export API](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)
