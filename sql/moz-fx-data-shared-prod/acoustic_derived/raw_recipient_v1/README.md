# Raw Recipient (Acoustic Data)

## Overview

This table imports raw recipient-level data exported from Acoustic Campaign, providing detailed email performance metrics at the individual contact level. The data includes comprehensive tracking of email events such as sends, opens, clicks, bounces, and unsubscribes, enabling granular analysis of email campaign effectiveness.

## Data Source

Data is exported from Acoustic Campaign via CSV files and ingested into BigQuery. The source API documentation can be found at: https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport

## Table Characteristics

- **Partition Field**: `submission_date` (day-level partitioning)
- **Clustering**: `event_type`, `recipient_type`, `body_type`
- **Incremental Load**: Yes
- **Retention**: 775 days
- **Table Type**: Client-level data

## Key Columns

- **recipient_id**: Unique identifier for each contact
- **mailing_id**: Links to specific email mailing campaigns
- **campaign_id**: Groups mailings under broader campaigns
- **event_type**: Categorizes recipient actions (sent, opened, clicked, etc.)
- **event_timestamp**: Precise timing of events in the API user's timezone
- **url**: Captures clicked links for engagement analysis
- **suppression_reason**: Tracks why contacts were excluded from mailings

## Downstream Analysis Suggestions

### 1. Email Campaign Performance Dashboard
Analyze open rates, click-through rates, and conversion funnels by campaign and content type. Calculate engagement metrics by comparing event types (sent → opened → clicked) to identify high-performing campaigns and optimize future email strategies.

### 2. Recipient Engagement Segmentation
Segment recipients based on engagement patterns using `event_type` and `recipient_type`. Identify highly engaged users (multiple clicks/opens), inactive users, and suppressed contacts to tailor re-engagement campaigns and improve list health.

### 3. Content and Link Performance Analysis
Analyze `click_name` and `url` fields to determine which content elements and links drive the most engagement. Compare performance across different `body_type` formats (HTML vs. plain text) to optimize email design and content placement.

### 4. Suppression and List Quality Monitoring
Track suppression trends over time using `suppression_reason` to identify potential deliverability issues, monitor unsubscribe rates, and maintain list hygiene. Cross-reference with campaign characteristics to understand factors driving opt-outs.

## Usage Notes

- The table is clustered by event-related fields for efficient filtering on common query patterns
- Partition filtering on `submission_date` is recommended for query performance
- Event timestamps are in the API user's timezone; consider timezone conversion for analysis across regions
- The data represents raw, unaggregated events and may require deduplication for certain analyses
