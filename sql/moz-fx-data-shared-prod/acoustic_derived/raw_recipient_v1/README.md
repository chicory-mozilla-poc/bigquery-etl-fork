# Raw Recipient (Acoustic Data)

## Overview

This table contains raw email event data exported from Acoustic Campaign, a marketing automation platform. It captures detailed metrics around email performance including sends, opens, clicks, bounces, and unsubscribes. The data is imported from CSV exports and provides contact-level event tracking.

## Table Details

- **Database**: moz-fx-data-shared-prod
- **Dataset**: acoustic_derived
- **Table**: raw_recipient_v1
- **Table Type**: Client-level
- **Partitioning**: Daily partitioning on `submission_date` (775 days retention)
- **Clustering**: `event_type`, `recipient_type`, `body_type`

## Data Source

Data is exported from Acoustic Campaign via their Raw Recipient Data Export API.

**API Reference**: https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport

## Key Fields

- **recipient_id**: Identifies the contact/recipient
- **mailing_id**: Links to the specific sent email
- **campaign_id**: Links to the marketing campaign
- **event_type**: Type of interaction (open, click, bounce, etc.)
- **event_timestamp**: When the event occurred
- **url**: For click events, the URL that was clicked
- **submission_date**: ETL partition field for efficient querying

## Downstream Analysis Suggestions

### 1. Email Campaign Performance Analysis
Analyze campaign effectiveness by aggregating event types (opens, clicks, bounces) by campaign_id and mailing_id. Calculate metrics like open rate, click-through rate (CTR), and bounce rate to identify high-performing campaigns and content strategies.

### 2. Recipient Engagement Segmentation
Segment recipients based on engagement patterns using event_type and event_timestamp. Identify highly engaged contacts (frequent opens/clicks), dormant contacts (no recent activity), and at-risk contacts (recent unsubscribes or bounces) to inform targeting strategies.

### 3. Content Performance and A/B Testing
Compare performance across different body_type and content_id combinations to understand which email formats and content variations drive better engagement. Analyze click_name and url patterns to identify which calls-to-action and links resonate most with recipients.

### 4. Email Deliverability and Suppression Analysis
Monitor suppression_reason trends over time to identify deliverability issues. Track bounce patterns, unsubscribe reasons, and recipient_type distributions to maintain list health and improve sender reputation.

## Owners

- cbeck@mozilla.com

## Notes

- The table uses incremental loading to append new data daily
- Data is partitioned by submission_date for efficient querying
- Clustering on event_type, recipient_type, and body_type optimizes common query patterns
- Event timestamps are in the API user's configured time zone
