# Raw Recipient (Acoustic Campaign Data)

## Overview

The `raw_recipient_v1` table contains raw, recipient-level email event data exported from Acoustic Campaign (formerly IBM Watson Campaign Automation). This table provides comprehensive tracking of all email interactions including sends, opens, clicks, bounces, opt-outs, and suppressions. The data is imported via CSV exports from the Acoustic Campaign API and represents email performance metrics at the individual contact level.

This table serves as the foundational dataset for analyzing email campaign performance, recipient engagement patterns, deliverability metrics, and contact behavior across Mozilla's email marketing operations managed through Acoustic Campaign.

**Key Characteristics:**
- **Data Source:** Acoustic Campaign API (CSV export)
- **Granularity:** Individual recipient-level email events
- **Update Pattern:** Incremental daily loads
- **Data Scope:** PII-removed contact and event data
- **Access Level:** Mozilla Confidential (restricted to mozilla-confidential workgroup)

## Schema

The table contains 13 fields organized into two categories:

### Email Event Fields (from Acoustic Campaign)

| Column | Type | Description |
|--------|------|-------------|
| `recipient_id` | INTEGER | Unique identifier for the email recipient/contact |
| `report_id` | INTEGER | Internal Acoustic Campaign report identifier |
| `mailing_id` | INTEGER | Unique identifier for the specific email send/mailing |
| `campaign_id` | INTEGER | Identifier for the parent marketing campaign |
| `content_id` | STRING | Identifier for the email content or creative variant |
| `event_timestamp` | DATETIME | Date and time when the email event occurred |
| `event_type` | STRING | Type of email interaction (Sent, Opens, Clicks, Opt-outs, Bounces, Suppressions) |
| `recipient_type` | STRING | Classification of the recipient (subscriber, test, seed list, etc.) |
| `body_type` | STRING | Format of email content (HTML, Text, Multipart) |
| `click_name` | STRING | User-defined label for clicked links (populated only for click events) |
| `url` | STRING | Full hyperlink URL clicked by recipient (populated only for click events) |
| `suppression_reason` | STRING | Reason code for email suppression (populated only for suppression events) |

### ETL Fields

| Column | Type | Description |
|--------|------|-------------|
| `submission_date` | DATE | ETL execution date and partition key (775-day retention) |

For detailed field descriptions, see [schema.yaml](schema.yaml).

## Data Sources

### Primary Source
- **System:** Acoustic Campaign (IBM Watson Campaign Automation)
- **API Documentation:** [Raw Recipient Data Export](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)
- **Export Format:** CSV file extracts
- **Data Owner:** cbeck@mozilla.com

### Data Flow
1. Email events generated in Acoustic Campaign as recipients interact with emails
2. CSV export files created via Acoustic Campaign API
3. Daily ETL process imports CSV data into BigQuery
4. Data partitioned by `submission_date` and clustered by `event_type`, `recipient_type`, and `body_type`

## Update Schedule

- **Frequency:** Daily incremental loads
- **Load Pattern:** Incremental (new events added daily)
- **Partition Field:** `submission_date` (daily partitions)
- **Data Retention:** 775 days (automatic partition expiration)
- **Expected Lag:** Events from the previous day loaded during daily Airflow execution

## Usage Examples

### Example 1: Email Open Rate by Campaign
```sql
SELECT
  campaign_id,
  mailing_id,
  COUNTIF(event_type = 'Sent') AS emails_sent,
  COUNTIF(event_type = 'Opens') AS emails_opened,
  SAFE_DIVIDE(
    COUNTIF(event_type = 'Opens'),
    COUNTIF(event_type = 'Sent')
  ) * 100 AS open_rate_pct
FROM
  `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE
  submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND event_type IN ('Sent', 'Opens')
GROUP BY
  campaign_id,
  mailing_id
ORDER BY
  open_rate_pct DESC;
```

### Example 2: Click-Through Analysis
```sql
SELECT
  click_name,
  url,
  COUNT(DISTINCT recipient_id) AS unique_clickers,
  COUNT(*) AS total_clicks
FROM
  `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE
  submission_date = CURRENT_DATE() - 1
  AND event_type = 'Clicks'
  AND click_name IS NOT NULL
GROUP BY
  click_name,
  url
ORDER BY
  total_clicks DESC;
```

### Example 3: Bounce and Suppression Analysis
```sql
SELECT
  suppression_reason,
  COUNT(DISTINCT recipient_id) AS affected_contacts,
  COUNT(*) AS total_occurrences
FROM
  `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE
  submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
  AND event_type = 'Suppressions'
  AND suppression_reason IS NOT NULL
GROUP BY
  suppression_reason
ORDER BY
  affected_contacts DESC;
```

### Example 4: Event Timeline for a Specific Mailing
```sql
SELECT
  event_timestamp,
  event_type,
  recipient_type,
  body_type,
  COUNT(DISTINCT recipient_id) AS recipient_count
FROM
  `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE
  mailing_id = 12345  -- Replace with actual mailing_id
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 14 DAY)
GROUP BY
  event_timestamp,
  event_type,
  recipient_type,
  body_type
ORDER BY
  event_timestamp;
```

## Performance Optimization

### Partitioning
- The table is partitioned by `submission_date` (daily)
- Partition filters are **recommended** but not required
- Always include date filters to reduce scan costs and improve performance
- Example: `WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)`

### Clustering
The table is clustered on three dimensions for optimal query performance:
1. **`event_type`** - Filter by specific event types (Sent, Opens, Clicks, etc.)
2. **`recipient_type`** - Segment by recipient categories
3. **`body_type`** - Analyze by email content format

Queries filtering on these fields benefit from reduced data scanning.

## Related Tables

### Acoustic Dataset Tables
- **`acoustic_derived.contact_v1`** - Contact/recipient profile data (joinable via `recipient_id`)
- **`acoustic_derived.contact_current_snapshot_v1`** - Current state snapshot of contacts

### Joining with Contact Data
```sql
SELECT
  r.mailing_id,
  r.event_type,
  c.email_address,  -- Example field from contact table
  COUNT(*) AS event_count
FROM
  `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1` r
LEFT JOIN
  `moz-fx-data-shared-prod.acoustic_derived.contact_v1` c
  ON r.recipient_id = c.recipient_id
WHERE
  r.submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
GROUP BY
  r.mailing_id,
  r.event_type,
  c.email_address;
```

## Notes

### Data Characteristics
- **NULL Handling:** Many fields are conditionally populated based on `event_type`
  - `click_name` and `url` only populated for click events
  - `suppression_reason` only populated for suppression events
- **Event Types:** Common values include Sent, Opens, Clicks, Opt-outs, Bounces, and Suppressions
- **PII Status:** This table is in the `acoustic_derived` dataset which holds data with PII removed
- **Time Zone:** `event_timestamp` reflects the API user's configured time zone in Acoustic

### Best Practices
1. **Always filter by `submission_date`** to reduce query costs
2. **Use clustering fields** (`event_type`, `recipient_type`, `body_type`) in WHERE clauses
3. **Join with contact tables** via `recipient_id` for enriched analysis
4. **Consider event_type** when interpreting other fields (some are event-specific)
5. **Monitor data freshness** - expect daily updates with previous day's events

### Access & Permissions
- Dataset: `moz-fx-data-shared-prod.acoustic_derived`
- Access Level: Mozilla Confidential
- Viewer Role: `workgroup:mozilla-confidential`
- Contact data owner for access requests: cbeck@mozilla.com

### Support & Documentation
- **Acoustic API Reference:** [Raw Recipient Data Export API](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)
- **Data Owner:** cbeck@mozilla.com
- **Dataset:** `acoustic_derived` (Acoustic tables without PII)
