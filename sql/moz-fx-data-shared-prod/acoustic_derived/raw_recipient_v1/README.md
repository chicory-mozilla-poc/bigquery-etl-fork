# Raw Recipient (Acoustic Data)

## Overview

This table contains raw recipient-level email engagement data exported from Acoustic Campaign via CSV imports. It captures detailed metrics around email performance including sends, opens, clicks, bounces, and suppressions for Mozilla's email campaigns.

**Source:** Acoustic Campaign Raw Recipient Data Export API
**Data Level:** Contact/recipient-level events
**Update Frequency:** Incremental daily imports
**Retention:** 775 days (partitioned by submission_date)

## Table Details

- **Project:** moz-fx-data-shared-prod
- **Dataset:** acoustic_derived
- **Table:** raw_recipient_v1
- **Partitioning:** Daily partition on `submission_date`
- **Clustering:** `event_type`, `recipient_type`, `body_type`

## Schema Summary

The table contains 13 columns capturing:
- **Identifiers:** recipient_id, mailing_id, campaign_id, content_id
- **Event Details:** event_timestamp, event_type
- **Recipient Classification:** recipient_type, body_type
- **Click Tracking:** click_name, url
- **Suppression Data:** suppression_reason
- **ETL Metadata:** submission_date

## Key Metrics Captured

- Email sends and deliveries
- Open events and engagement timing
- Click-through events with link details
- Bounce and suppression reasons
- Recipient segmentation by type

## Downstream Analysis Suggestions

### 1. Email Campaign Performance Dashboard
Aggregate metrics by `campaign_id` and `mailing_id` to calculate:
- Open rates: `COUNT(DISTINCT CASE WHEN event_type = 'open' THEN recipient_id END) / COUNT(DISTINCT CASE WHEN event_type = 'sent' THEN recipient_id END)`
- Click-through rates (CTR) by campaign
- Time-to-open analysis using `event_timestamp`
- Compare performance across `body_type` (HTML vs text)

### 2. Link Performance Analysis
Analyze click behavior by:
- Most clicked URLs and `click_name` labels
- Click patterns by recipient segment (`recipient_type`)
- URL engagement trends over time
- A/B testing different link placements or content variants (`content_id`)

### 3. Suppression and Deliverability Monitoring
Track email health metrics:
- Suppression trends by `suppression_reason`
- Bounce rate analysis by campaign
- Recipient type distribution to identify seed contacts
- Weekly/monthly deliverability scorecards

### 4. Recipient Engagement Cohort Analysis
Segment recipients by engagement patterns:
- Identify highly engaged vs dormant contacts based on `event_type` frequency
- Time-based cohorts using first `event_timestamp`
- Cross-campaign engagement analysis by `recipient_id`
- Predict churn risk based on declining engagement

## Important Notes

- The `submission_date` field represents the ETL execution date and should align with dates in `event_timestamp`
- Partition filters on `submission_date` are recommended for query performance
- Multiple event types per recipient per mailing are expected (e.g., sent → opened → clicked)
- The table uses clustering on event dimensions for optimized filtering

## Owners

- cbeck@mozilla.com

## References

- [Acoustic Campaign Raw Recipient Data Export API](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)
