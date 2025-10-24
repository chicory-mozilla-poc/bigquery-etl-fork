# Raw Recipient (Acoustic Data)

## Table Overview

**Table Name:** `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`

This table contains raw recipient-level event data exported from Acoustic Campaign as CSV files. It captures comprehensive email performance metrics and recipient interactions, including clicks, opens, bounces, and other engagement events. The data is imported incrementally and partitioned by submission date for efficient querying.

**Table Type:** Client-level
**Incremental:** Yes
**Time Partitioning:** Daily (on `submission_date` field)
**Clustering:** By `event_type`, `recipient_type`, `body_type`

## Data Source

Data is imported from Acoustic Campaign's Raw Recipient Data Export API.
Reference: [Acoustic Campaign API Documentation](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)

## Schema

| Column Name | Type | Description |
|------------|------|-------------|
| `recipient_id` | INTEGER | Unique identifier for the contact who received or interacted with the email campaign. |
| `report_id` | INTEGER | Report identifier associated with the email campaign metrics and analytics. |
| `mailing_id` | INTEGER | Unique identifier for the specific sent email mailing associated with this recipient event. |
| `campaign_id` | INTEGER | Campaign identifier linking this recipient event to its parent marketing campaign. |
| `content_id` | STRING | Content identifier for the specific email template or message version sent to the recipient. |
| `event_timestamp` | DATETIME | Timestamp when the recipient event occurred, recorded in the API user's configured time zone. |
| `event_type` | STRING | Type of recipient interaction event (e.g., sent, opened, clicked, bounced, unsubscribed). |
| `recipient_type` | STRING | Classification of the recipient contact type in the Acoustic Campaign system. |
| `body_type` | STRING | Format of the email body received by the contact (e.g., HTML, plain text, multipart). |
| `click_name` | STRING | User-defined name assigned to the tracked link or clickstream element within the email. |
| `url` | STRING | Full URL of the hyperlink that was clicked by the recipient in a clickthrough or clickstream event. |
| `suppression_reason` | STRING | Reason why a contact was suppressed from receiving emails (e.g., bounced, unsubscribed, opted out). |
| `submission_date` | DATE | Date partition field representing the Airflow execution date, used for data organization and filtering. |

## Downstream Analysis Suggestions

### 1. Email Campaign Performance Analysis
Analyze email engagement rates across different campaigns by examining `event_type` distributions (opens, clicks, bounces) grouped by `campaign_id` and `mailing_id`. Calculate key metrics like open rate, click-through rate (CTR), and bounce rate to identify high-performing campaigns and optimize future email strategies.

### 2. Content Effectiveness Comparison
Compare the performance of different email body types (`body_type`) and content versions (`content_id`) to determine which formats drive better engagement. Analyze click patterns by `click_name` and `url` to identify the most effective calls-to-action and content placements.

### 3. Recipient Segmentation and Behavior Patterns
Segment recipients by `recipient_type` and analyze their engagement patterns over time using `event_timestamp`. Identify high-value recipient segments, optimal send times, and behavioral trends to improve targeting and personalization strategies.

### 4. Email Deliverability and Suppression Analysis
Monitor email deliverability health by analyzing `suppression_reason` patterns and bounce rates. Track suppression trends over time to identify potential delivery issues, maintain list hygiene, and ensure compliance with email best practices.

## Owners

- cbeck@mozilla.com

## Additional Information

- **Partition Expiration:** 775 days
- **Requires Partition Filter:** No
- **Labels:** incremental, client_level
