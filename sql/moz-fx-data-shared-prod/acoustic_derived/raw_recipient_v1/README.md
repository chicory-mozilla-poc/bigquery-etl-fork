# Raw Recipient (Acoustic Campaign Data)

## Overview

This table contains raw email recipient engagement data exported from Acoustic Campaign via their API. It captures granular email performance metrics including sends, opens, clicks, bounces, and suppressions at the recipient level. The data is imported directly from CSV exports provided by Acoustic Campaign and serves as the foundational dataset for email marketing analytics.

## Data Source

- **Platform**: Acoustic Campaign (formerly IBM Watson Campaign Automation)
- **API Reference**: [Raw Recipient Data Export API](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)
- **Update Frequency**: Daily incremental loads via Airflow
- **Data Retention**: 775 days (partitioned by submission_date)

## Table Structure

### Partitioning
- **Type**: Day-based partitioning
- **Field**: `submission_date`
- **Partition Filter**: Not required (but recommended for query performance)

### Clustering
Optimized for queries filtering on:
- `event_type` - Email engagement event types
- `recipient_type` - Recipient classification
- `body_type` - Email format types

## Key Columns

| Column | Type | Description |
|--------|------|-------------|
| `recipient_id` | INTEGER | Unique contact identifier |
| `mailing_id` | INTEGER | Specific email send identifier |
| `campaign_id` | INTEGER | Campaign identifier |
| `event_timestamp` | DATETIME | Event occurrence timestamp |
| `event_type` | STRING | Type of engagement (sent, opened, clicked, etc.) |
| `url` | STRING | Clicked link URL (for click events) |
| `suppression_reason` | STRING | Reason for email suppression |
| `submission_date` | DATE | ETL processing date for partitioning |

## Downstream Analysis Suggestions

### 1. Email Campaign Performance Dashboard
Calculate key email metrics by campaign:
- **Open Rate**: Ratio of unique opens to emails sent
- **Click-Through Rate (CTR)**: Ratio of unique clicks to emails sent
- **Click-to-Open Rate (CTOR)**: Ratio of clicks to opens
- **Bounce Rate**: Percentage of failed deliveries
- **Unsubscribe Rate**: Opt-out rate per campaign

Group by `campaign_id` and `event_type`, joining with campaign metadata for rich reporting.

### 2. Recipient Engagement Segmentation
Identify high-value and at-risk segments:
- **Engaged Users**: Recipients with consistent open/click activity
- **Inactive Users**: Recipients with declining engagement trends
- **Single-Click Champions**: Users who click but don't convert
- **Body Type Preferences**: Analyze engagement patterns by HTML vs. plain text

Use `recipient_id` with time-based aggregations of `event_type` and `body_type`.

### 3. Content Performance Analysis
Evaluate which email content drives engagement:
- Compare click-through rates across different `content_id` values
- Identify top-performing links by analyzing `url` and `click_name`
- A/B test analysis when multiple content variants exist per campaign
- Determine optimal send times by analyzing `event_timestamp` patterns

Aggregate by `content_id`, `url`, and `click_name` to surface winning content strategies.

### 4. Email Deliverability Health Monitoring
Track deliverability issues and list hygiene:
- Monitor bounce rates and types over time
- Analyze `suppression_reason` patterns to identify systemic issues
- Calculate inbox placement trends (sent vs. bounced)
- Flag campaigns with unusually high suppression rates

Filter by suppression-related `event_type` values and aggregate by `submission_date` for trending.

## Data Quality Notes

- All fields are nullable, so handle null values appropriately in queries
- `event_timestamp` is in the API user's configured timezone
- Multiple event types can exist per recipient per mailing (e.g., sent → opened → clicked)
- The `report_id` field currently lacks detailed documentation in the source schema

## Related Tables

Consider joining with:
- Campaign metadata tables for campaign names and attributes
- Recipient profile tables for demographic enrichment
- Conversion/revenue tables for ROI analysis

## Owner

- **Contact**: cbeck@mozilla.com
