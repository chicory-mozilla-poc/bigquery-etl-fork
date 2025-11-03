# Raw Recipient (Acoustic Campaign Data)

## Overview

The `raw_recipient_v1` table contains raw recipient-level event data exported from Acoustic Campaign platform. This table captures comprehensive email engagement metrics including sends, opens, clicks, bounces, and suppressions. The data is imported directly from Acoustic's CSV export and provides granular tracking of email campaign performance at the individual recipient level.

**Data Source**: Acoustic Campaign API
**Update Frequency**: Incremental daily loads
**Retention**: 775 days (partitioned by submission_date)

## Table Details

- **Dataset**: `moz-fx-data-shared-prod.acoustic_derived`
- **Table**: `raw_recipient_v1`
- **Type**: Client-level data
- **Partitioning**: Daily partitioned by `submission_date`
- **Clustering**: `event_type`, `recipient_type`, `body_type`

## Schema

| Column | Type | Description |
|--------|------|-------------|
| `recipient_id` | INTEGER | Unique identifier for the contact who received the email campaign |
| `report_id` | INTEGER | Report identifier used for tracking and aggregating campaign metrics |
| `mailing_id` | INTEGER | Unique identifier for the specific sent email mailing |
| `campaign_id` | INTEGER | Campaign identifier linking events to their parent marketing campaign |
| `content_id` | STRING | Identifier for the email content variant sent to the recipient |
| `event_timestamp` | DATETIME | Timestamp of when the email event occurred in the API user's timezone |
| `event_type` | STRING | Type of email engagement event (e.g., sent, opened, clicked, bounced) |
| `recipient_type` | STRING | Classification of the recipient type who received the campaign email |
| `body_type` | STRING | Format of the email body delivered to the recipient (HTML, text, or multipart) |
| `click_name` | STRING | User-defined label for the clicked link or clickstream tracking element |
| `url` | STRING | Full URL of the hyperlink clicked by the recipient |
| `suppression_reason` | STRING | Reason why a contact was excluded from receiving the email (e.g., unsubscribed, bounced) |
| `submission_date` | DATE | Date partition field representing when data was loaded into BigQuery |

## Downstream Analysis Suggestions

### 1. Campaign Performance Dashboard
Aggregate email engagement metrics by campaign to calculate:
- Open rates: `COUNT(DISTINCT CASE WHEN event_type = 'open' THEN recipient_id END) / COUNT(DISTINCT CASE WHEN event_type = 'sent' THEN recipient_id END)`
- Click-through rates: `COUNT(DISTINCT CASE WHEN event_type = 'click' THEN recipient_id END) / COUNT(DISTINCT CASE WHEN event_type = 'sent' THEN recipient_id END)`
- Bounce rates by campaign and recipient type
- Time-series trends of engagement over submission_date

### 2. Recipient Engagement Segmentation
Analyze recipient behavior patterns:
- Segment recipients by engagement level (high, medium, low) based on click and open frequency
- Identify most engaged recipient_type segments
- Track recipient lifecycle from first send to potential suppression
- Compare engagement rates across different body_type formats (HTML vs text)

### 3. Content Performance Analysis
Evaluate content effectiveness:
- Rank content_id variants by engagement metrics
- Analyze click_name and url patterns to identify high-performing CTAs
- Compare mailing_id performance within the same campaign
- Identify optimal send times by analyzing event_timestamp patterns

### 4. Suppression and Deliverability Analysis
Monitor email health metrics:
- Track suppression_reason trends over time to identify list quality issues
- Calculate deliverability rates by recipient_type
- Identify campaigns with unusually high suppression or bounce rates
- Correlate suppression patterns with campaign characteristics

## Data Quality Notes

- The `event_timestamp` field uses the API user's timezone, not UTC
- Records are partitioned by `submission_date` which should overlap with dates in `event_timestamp`
- Some fields like `click_name` and `url` are only populated for click events
- The `suppression_reason` field is only populated when contacts are suppressed

## Owners

- cbeck@mozilla.com

## References

- [Acoustic Campaign API Documentation](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)
