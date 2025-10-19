# clients_daily_v6

## Overview

The `clients_daily_v6` table provides a comprehensive daily aggregation of Firefox Desktop telemetry data at the client level. This table serves as a foundational dataset for understanding user behavior, browser configuration, system characteristics, and engagement metrics across the Firefox user base.

## Table-Level Quality Description

**Purpose**: This table aggregates Firefox Desktop telemetry data from the `telemetry_stable.main_v5` source table, providing one row per client per day with summarized metrics and configuration information. It is designed to support analysis of user engagement, feature adoption, system compatibility, performance issues, and browser health.

**Data Sources**:
- Primary Source: `moz-fx-data-shared-prod.telemetry_stable.main_v5`
- Filters Applied:
  - `normalized_app_name = 'Firefox'` (Desktop Firefox only)
  - `document_id IS NOT NULL` (Valid telemetry pings)
  - Excludes overactive clients (>150,000 pings/day or >3,000,000 active addons aggregate)

**Update Schedule**:
- **Frequency**: Daily
- **Partition Key**: `submission_date` (DATE)
- **Processing Time**: Typically completes within 2-4 hours after midnight UTC
- **Data Freshness**: Contains data from the previous day (D-1)

**Key Characteristics**:
- **Grain**: One row per `client_id` per `submission_date`
- **Aggregation Logic**:
  - Uses `mode_last` (most recent value) for configuration settings
  - Uses `SUM` for cumulative metrics (searches, crashes, active hours)
  - Uses `AVG` for rate-based metrics (clock skew, bookmarks count)
  - Aggregates histogram data from multiple pings per client per day
- **Data Quality Controls**:
  - Filters out overactive clients to prevent aggregation errors
  - Handles NULL values appropriately for optional fields
  - Preserves data type integrity through safe casting
- **Historical Context**: Version 6 (v6) reflects schema evolution from earlier versions, with additional columns for newer Firefox features