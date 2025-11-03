# Raw Recipient (Acoustic Campaign Data)

## Overview

This table contains raw recipient-level event data exported from Acoustic Campaign. It captures detailed email interaction metrics including sends, opens, clicks, bounces, and suppressions. The data is imported from CSV files provided by Acoustic Campaign's Raw Recipient Data Export API.

**Table:** `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`

## Data Source

Data is sourced from [Acoustic Campaign Raw Recipient Data Export API](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)

## Table Properties

- **Type:** Client-level data
- **Update Frequency:** Incremental
- **Time Partitioning:** Daily, partitioned on `submission_date`
- **Clustering:** `event_type`, `recipient_type`, `body_type`
- **Data Retention:** 775 days

## Schema Summary

The table contains 13 columns tracking:
- **Recipient Information:** Contact identifiers and recipient types
- **Campaign Context:** Mailing, campaign, and content IDs
- **Event Details:** Event types, timestamps, and interaction data
- **Click Tracking:** Link names and URLs for click-through analysis
- **Delivery Status:** Suppression reasons for failed deliveries
- **ETL Metadata:** Processing dates for data pipeline tracking

## Key Metrics & Events

The `event_type` field captures various email lifecycle events:
- Email sends and deliveries
- Email opens (unique and total)
- Link clicks and clickstreams
- Bounces (hard and soft)
- Unsubscribes and opt-outs
- Suppressions

## Downstream Analysis Suggestions

### 1. Email Engagement Analysis
Analyze open rates, click-through rates, and engagement patterns by campaign, content, and recipient type. Calculate metrics like:
- Open rate: opens / sends
- Click-through rate: clicks / opens
- Conversion funnel: sent → opened → clicked

### 2. Campaign Performance Comparison
Compare campaign effectiveness across different mailings using `campaign_id` and `mailing_id`. Identify top-performing content by analyzing `content_id` with engagement metrics.

### 3. Deliverability Health Monitoring
Track email deliverability issues by analyzing `suppression_reason` patterns over time. Monitor bounce rates and identify problematic recipient segments or domains.

### 4. Time-Series Trend Analysis
Leverage `event_timestamp` and `submission_date` to analyze email performance trends, identify optimal send times, and detect anomalies in engagement patterns across different time periods.

## Owners

- cbeck@mozilla.com

## Notes

- Events are recorded in the API user's timezone
- The `submission_date` should align with dates within the `event_timestamp` field
- Table uses time partitioning for efficient querying - consider using partition filters for better performance
