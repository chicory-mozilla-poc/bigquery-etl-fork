# Raw Recipient - Acoustic Campaign Data

## Table Overview

**Table**: `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`

This table contains raw recipient-level event data exported from Acoustic Campaign (formerly IBM Watson Campaign Automation). It captures detailed email performance metrics and recipient interactions including opens, clicks, bounces, and suppressions. The data is imported from CSV exports and provides granular tracking of individual contact behavior across email campaigns.

**Key Features**:
- Recipient-level event tracking for all email campaign interactions
- Time-partitioned by submission date with 775-day retention
- Clustered by event_type, recipient_type, and body_type for optimized query performance
- Incremental loading pattern with daily updates

## Owners

- **cbeck@mozilla.com**

## Schema Details

The table includes 13 columns tracking:
- **Recipient identifiers**: recipient_id, mailing_id, campaign_id
- **Event details**: event_type, event_timestamp, body_type
- **Engagement tracking**: click_name, url
- **Email management**: suppression_reason, recipient_type
- **Content reference**: content_id, report_id
- **ETL metadata**: submission_date

## Downstream Analysis Suggestions

### 1. Email Campaign Performance Dashboard
Analyze email engagement rates by aggregating event_type (opens, clicks, bounces) grouped by campaign_id and mailing_id. Calculate key metrics like:
- Open rate: COUNT(DISTINCT recipient_id WHERE event_type = 'open') / COUNT(DISTINCT recipient_id WHERE event_type = 'sent')
- Click-through rate (CTR): clicks / opens
- Bounce rate: bounces / sent
- Track trends over time using event_timestamp

### 2. Recipient Engagement Segmentation
Segment recipients based on their interaction patterns across multiple campaigns. Identify:
- Highly engaged users (multiple opens/clicks across campaigns)
- At-risk users (no recent engagement)
- Suppressed recipients and suppression_reason distribution
- Engagement patterns by recipient_type and body_type preferences

### 3. Link Performance Analysis
Evaluate which links and CTAs drive the most engagement by analyzing:
- Top performing click_name and url combinations
- Click patterns by campaign_id to identify successful content strategies
- A/B testing results by comparing different content_id versions
- Clickstream analysis to understand user journey through emails

### 4. Email Deliverability & Health Monitoring
Monitor email program health and identify deliverability issues:
- Suppression trends over time and dominant suppression_reason categories
- Body_type performance comparison (HTML vs text vs multipart)
- Temporal patterns in event_timestamp to optimize send times
- Campaign-level health scores combining delivery, engagement, and suppression metrics

## Data Source

Data is exported from Acoustic Campaign via their Raw Recipient Data Export API.
Reference: https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport

## Table Configuration

- **Partitioning**: Daily partitions on `submission_date`
- **Clustering**: `event_type`, `recipient_type`, `body_type`
- **Retention**: 775 days
- **Update Frequency**: Daily incremental loads
