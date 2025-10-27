# FxA Content Events v1

## Overview

**⚠️ DEPRECATED TABLE - NO LONGER ACTIVELY MAINTAINED**

This table contains selected Amplitude events extracted from Firefox Accounts (FxA) content server logs. The table is **no longer scheduled** for updates as the underlying source tables from the FxA side have been removed. FxA content server events are now included in the `fxa_gcp_stdout_events_v1` ETL pipeline instead.

Historical data in this table is still relevant and referenced in downstream views. If you need current FxA content events, please use `moz-fx-data-shared-prod.firefox_accounts_derived.fxa_gcp_stdout_events_v1` instead.

**Table:** `moz-fx-data-shared-prod.firefox_accounts_derived.fxa_content_events_v1`  
**Owner:** kik@mozilla.com  
**Application:** Firefox Accounts  
**Table Type:** Client-level events  
**Status:** Deprecated (historical data only)

## Purpose

This table was designed to capture user behavior and analytics events from the Firefox Accounts content server, which handles user authentication, registration, and account management flows. The data was processed through Amplitude analytics to track:

- User registration and login flows
- Account management activities
- Device connection and synchronization events
- Marketing attribution (UTM parameters)
- Error tracking and Content Security Policy violations

## Schema

The table follows the Google Cloud Platform (GCP) logging structure with nested JSON payloads containing Amplitude event data. Key top-level fields:

| Field | Type | Description |
|-------|------|-------------|
| `timestamp` | TIMESTAMP | GCP log timestamp (used for partitioning) |
| `jsonPayload` | RECORD | Primary data container with Amplitude events |
| `httpRequest` | RECORD | HTTP request metadata |
| `resource` | RECORD | GCP resource information |
| `severity` | STRING | Log severity level |

### Critical Fields in jsonPayload

| Field | Type | Description |
|-------|------|-------------|
| `jsonPayload.fields.event_type` | STRING | **Required** - Amplitude event type (e.g., 'fxa_login - view') |
| `jsonPayload.user_id` | STRING | **SHA256 hashed** - User identifier for privacy |
| `jsonPayload.device_id` | STRING | **SHA256 hashed** - Device identifier for analytics |
| `jsonPayload.flow_id` | STRING | Flow tracking identifier for user journeys |
| `jsonPayload.event_properties` | RECORD | Event-specific contextual properties |
| `jsonPayload.user_properties` | RECORD | User-level properties for analytics |
| `jsonPayload.service` | STRING | Mozilla service initiating the interaction |

### Privacy Features

User identifiers are protected using SHA256 hashing:
```sql
TO_HEX(SHA256(jsonPayload.fields.user_id)) AS user_id
TO_HEX(SHA256(jsonPayload.fields.device_id)) AS device_id
```

## Data Sources

**Primary Source (Deprecated):**
- `moz-fx-fxa-prod-0712.fxa_prod_logs.docker_fxa_content_20*` (sharded tables)
  - **Status:** Removed by FxA team
  - **Format:** Google Cloud Platform logs from FxA content server Docker containers
  - **Event Type Filter:** `jsonPayload.type = 'amplitudeEvent'`
  - **Required Field:** `jsonPayload.fields.event_type IS NOT NULL`

**Current Source:**
- FxA content events are now available in: `moz-fx-data-shared-prod.firefox_accounts_derived.fxa_gcp_stdout_events_v1`

## Update Schedule

**Historical Schedule:**
- **Frequency:** Daily
- **DAG:** `bqetl_fxa_events`
- **Incremental:** Yes
- **Partitioning:** Daily by `timestamp` field

**Current Status:**
- ❌ No longer scheduled
- ❌ Source tables removed
- ✅ Historical data preserved for backfill purposes

## Usage Examples

### ⚠️ For Historical Analysis Only

**Example 1: User Registration Events**
```sql
SELECT
  DATE(timestamp) AS event_date,
  jsonPayload.fields.event_type,
  jsonPayload.country,
  jsonPayload.service,
  COUNT(*) AS event_count
FROM
  `moz-fx-data-shared-prod.firefox_accounts_derived.fxa_content_events_v1`
WHERE
  DATE(timestamp) BETWEEN '2023-01-01' AND '2023-12-31'
  AND jsonPayload.fields.event_type LIKE 'fxa_reg%'
GROUP BY
  event_date, jsonPayload.fields.event_type, jsonPayload.country, jsonPayload.service
ORDER BY
  event_date DESC, event_count DESC
```

**Example 2: Flow Analysis**
```sql
SELECT
  jsonPayload.flow_id,
  jsonPayload.entrypoint,
  MIN(timestamp) AS flow_start,
  MAX(timestamp) AS flow_end,
  COUNT(*) AS events_in_flow,
  ARRAY_AGG(jsonPayload.fields.event_type ORDER BY timestamp) AS event_sequence
FROM
  `moz-fx-data-shared-prod.firefox_accounts_derived.fxa_content_events_v1`
WHERE
  DATE(timestamp) = '2023-06-15'
  AND jsonPayload.flow_id IS NOT NULL
GROUP BY
  jsonPayload.flow_id, jsonPayload.entrypoint
HAVING
  events_in_flow > 1
LIMIT 100
```

**Example 3: Marketing Attribution**
```sql
SELECT
  jsonPayload.utm_source,
  jsonPayload.utm_medium,
  jsonPayload.utm_campaign,
  COUNT(DISTINCT jsonPayload.user_id) AS unique_users,
  COUNT(*) AS total_events
FROM
  `moz-fx-data-shared-prod.firefox_accounts_derived.fxa_content_events_v1`
WHERE
  DATE(timestamp) BETWEEN '2023-01-01' AND '2023-12-31'
  AND jsonPayload.utm_source IS NOT NULL
GROUP BY
  jsonPayload.utm_source, jsonPayload.utm_medium, jsonPayload.utm_campaign
ORDER BY
  unique_users DESC
```

## Related Tables

### Current (Recommended)
- **`moz-fx-data-shared-prod.firefox_accounts_derived.fxa_gcp_stdout_events_v1`** - Current FxA content events (actively maintained)

### Historical/Deprecated
- `moz-fx-data-shared-prod.firefox_accounts_derived.fxa_content_events_v1` (this table)

### Related FxA Tables
- `moz-fx-data-shared-prod.firefox_accounts_derived.fxa_auth_events_v1` - FxA auth server events
- `moz-fx-data-shared-prod.firefox_accounts.fxa_all_events` - Unified FxA events view
- `moz-fx-data-shared-prod.firefox_accounts.fxa_users_services_daily` - Daily user service aggregations

## Notes

### Important Considerations

1. **Deprecation Status**
   - This table is **no longer updated** as of the source table removal
   - Historical data remains available for backfilling downstream ETL
   - Views may still reference this table for historical continuity

2. **Migration Information**
   - **Old Source:** `moz-fx-fxa-prod-0712.fxa_prod_logs.docker_fxa_content_20*`
   - **New Pipeline:** `fxa_gcp_stdout_events_v1`
   - **Reason:** FxA team removed underlying tables (FXA-6593)

3. **Known Issues**
   - The partitioned version of the source table had data completeness issues
   - Query uses sharded tables (`_TABLE_SUFFIX`) as a workaround
   - See query.sql comments for details on partitioning issue

4. **Privacy & Compliance**
   - User IDs and device IDs are SHA256 hashed
   - IP addresses are available in `httpRequest.remoteIp` for geolocation
   - UTM parameters track marketing attribution

5. **Data Quality**
   - All records have `jsonPayload.type = 'amplitudeEvent'`
   - Records without `event_type` are filtered out
   - Date filter uses `_TABLE_SUFFIX` format: `YYMMDD`

6. **For Current Analysis**
   - **Use:** `fxa_gcp_stdout_events_v1` for any analysis beyond historical backfills
   - **Contact:** kik@mozilla.com for questions about FxA event data

### Schema Complexity

The schema contains:
- **512 total fields** including nested structures
- **Multiple levels of nesting** (up to 4 levels deep)
- **Duplicate field names** at different nesting levels (e.g., `event`, `user_id`, `device_id`)
- **JSON-encoded strings** requiring additional parsing (e.g., `event_properties`, `user_properties` in fields record)

### Querying Tips

1. Always filter by date to optimize partition pruning: `DATE(timestamp) = @date`
2. Use fully qualified field paths to avoid ambiguity: `jsonPayload.fields.event_type`
3. Parse JSON strings when needed: `JSON_EXTRACT(jsonPayload.fields.event_properties, '$.service')`
4. Consider session-level aggregation using `jsonPayload.session_id`
5. Use `jsonPayload.flow_id` for end-to-end journey analysis

---

**Last Updated:** 2025-01-19  
**Documentation Version:** 1.0  
**Table Version:** v1 (deprecated)
