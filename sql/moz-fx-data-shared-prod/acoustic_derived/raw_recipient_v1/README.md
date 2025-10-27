# Raw Recipient Data (Acoustic Campaign)

## Overview

This table contains raw email recipient event data exported from Acoustic Campaign (formerly IBM Watson Campaign Automation). It captures detailed metrics and interactions related to email campaigns, including sends, opens, clicks, bounces, and suppressions.

## Data Source

- **Source System**: Acoustic Campaign API
- **API Reference**: [Raw Recipient Data Export](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)
- **Update Frequency**: Incremental daily loads
- **Partitioning**: Daily partitions by `submission_date` with 775-day retention

## Table Schema

The table includes:
- **Contact Identifiers**: `recipient_id`, `report_id`
- **Campaign Tracking**: `mailing_id`, `campaign_id`, `content_id`
- **Event Details**: `event_timestamp`, `event_type`, `recipient_type`, `body_type`
- **Click Tracking**: `click_name`, `url`
- **Suppression Data**: `suppression_reason`
- **ETL Metadata**: `submission_date`

## Clustering

The table is clustered by:
- `event_type` - For efficient filtering by interaction type
- `recipient_type` - For segmenting by recipient classification
- `body_type` - For analyzing email format performance

## Downstream Analysis Suggestions

### 1. Email Engagement Funnel Analysis
Analyze the progression from sends to opens to clicks by calculating conversion rates across `event_type` values. Compare performance across different `body_type` (HTML vs. text) and `recipient_type` segments to identify the most effective email formats and audience segments.

### 2. Campaign Performance Dashboard
Aggregate metrics by `campaign_id` and `mailing_id` to create comprehensive campaign performance reports. Track key metrics like open rates, click-through rates, and suppression rates over time using `submission_date` to identify trends and optimize future campaigns.

### 3. Link Click Analysis
Join `click_name` and `url` data to understand which links and content types drive the most engagement. Analyze click patterns by recipient segments to inform content strategy and optimize email CTAs and link placement.

### 4. Suppression and Deliverability Monitoring
Track `suppression_reason` trends over time to identify deliverability issues, manage list hygiene, and reduce bounce rates. Correlate suppression patterns with campaign characteristics to improve targeting and reduce opt-outs.

## Usage Notes

- Use `submission_date` for partition filtering to optimize query performance
- `event_timestamp` reflects the actual event time in the API user's timezone
- Multiple event types per recipient are expected (sent, opened, clicked, etc.)
- NULL values in `suppression_reason` indicate non-suppressed recipients
- Contact the owner for questions about data quality or access: cbeck@mozilla.com
