# Win10 Users Sync

## Overview

The `win10_users_sync_v1` table is designed to synchronize inactive Windows 10 Firefox users to Braze for the Win10 Inactive User Campaign. This table aggregates user profile data from Firefox Account (FxA) sources and prepares it in a format compatible with Braze's API for targeted user re-engagement campaigns.

## Table Description

This table transforms and packages user data from the `braze_derived.fxa_win10_users_historical_v1` source table into a JSON payload structure required by Braze. The table runs incrementally, processing only new records for each submission date to maintain an up-to-date sync with Braze.

## Schema

| Column | Type | Description |
|--------|------|-------------|
| `updated_at` | TIMESTAMP | Most recent date and time when the row was updated |
| `external_id` | STRING | Unique identifier for the user in external systems, typically a hashed UUID |
| `payload` | STRING | JSON payload containing user profile data for Braze API sync including email, locale, subscription preferences, and FxA identifier |

## Data Pipeline

The ETL process:
1. Extracts user data from `fxa_win10_users_historical_v1` for a specific submission date
2. Creates a JSON payload with structured user attributes including:
   - Email address
   - Email subscription status (set to "subscribed")
   - Subscription group preferences (UUID: `718eea53-371c-4cc6-9fdc-1260b1311bd8`)
   - User locale
   - FxA user identifier (SHA256 hashed)
3. Assigns a current timestamp for tracking sync freshness

## Payload Structure

The JSON payload contains the following structure:
```json
{
  "email": "user@example.com",
  "email_subscribe": "subscribed",
  "subscription_groups": [
    {
      "subscription_group_id": "718eea53-371c-4cc6-9fdc-1260b1311bd8",
      "subscription_state": "subscribed"
    }
  ],
  "locale": "en-US",
  "fxa_id_sha256": "abc123..."
}
```

## Scheduling

- **DAG Name:** `bqetl_braze_win10_sync`
- **Incremental:** Yes
- **Partition Parameter:** None (destination is the whole table, not a single partition)

## Downstream Analysis Suggestions

1. **Campaign Effectiveness Analysis**: Join with Braze campaign response data to measure re-engagement rates by locale, comparing email open rates and click-through rates across different language segments to optimize localization strategies.

2. **Subscription Preference Monitoring**: Track changes in subscription_groups over time to identify patterns in user opt-in/opt-out behavior, enabling proactive campaign adjustments for at-risk user segments.

3. **Sync Performance Metrics**: Analyze the volume of users synced per submission_date and the time lag between updated_at timestamps to ensure timely data delivery to Braze and identify potential bottlenecks in the sync pipeline.

4. **User Profile Completeness**: Evaluate the data quality by examining null patterns in the payload JSON fields (particularly email and locale completeness) to identify opportunities for improving data collection from upstream FxA sources.

## Owner

- lmcfall@mozilla.com
