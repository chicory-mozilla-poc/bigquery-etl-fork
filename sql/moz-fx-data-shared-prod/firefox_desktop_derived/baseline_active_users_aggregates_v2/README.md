# Baseline Active Users Aggregates v2

## Overview

The `baseline_active_users_aggregates_v2` table provides daily aggregated metrics of Firefox Desktop active users, derived from the baseline ping telemetry data. This table serves as a critical resource for tracking user engagement patterns, measuring daily/weekly/monthly active users (DAU/WAU/MAU), and analyzing user behavior across multiple dimensions including geographic location, application version, operating system, and attribution sources.

**Database**: `moz-fx-data-shared-prod`  
**Dataset**: `firefox_desktop_derived`  
**Table**: `baseline_active_users_aggregates_v2`

## Purpose

This aggregated table transforms individual user-level activity data into counts grouped by key dimensions, enabling efficient analysis of:
- User engagement trends (DAU/WAU/MAU)
- Geographic distribution of Firefox Desktop users
- Application version adoption rates
- Operating system usage patterns
- Marketing attribution effectiveness
- Default browser adoption

The table distinguishes between:
- **Users**: Counts based on ping submission (daily_users, weekly_users, monthly_users)
- **Active Users**: Counts based on actual browsing activity (dau, wau, mau)

## Data Sources

**Primary Source**: `moz-fx-data-shared-prod.firefox_desktop.baseline_active_users`

The query aggregates user-level boolean flags (`is_daily_user`, `is_weekly_user`, `is_monthly_user`, `is_dau`, `is_wau`, `is_mau`) into counts using the `COUNTIF` function, grouped by 24 dimensional attributes.

## Schema

### Temporal Dimensions
- **submission_date** (DATE): The date when the telemetry ping was received on the server side. Used for partitioning.
- **first_seen_year** (INTEGER): Year extracted from the first_seen_date, representing when the user's first ping was received. Enables cohort analysis.

### Geographic Dimensions
- **country** (STRING): ISO country code where the activity occurred, determined by IP geolocation. Unknown values stored as '??'.
- **city** (STRING): City reported by the client based on IP geolocation.

### Application Dimensions
- **app_name** (STRING): Application identifier (Firefox Desktop). Appended with distribution_id for MozillaOnline distributions and isp for BrowserStack clients.
- **app_version** (STRING): User-visible version string (e.g., "1.0.3") for the browser.
- **app_version_major** (NUMERIC): Major version number (e.g., 123 for version 123.0).
- **app_version_minor** (NUMERIC): Minor version number (e.g., 0 for version 123.0).
- **app_version_patch_revision** (NUMERIC): Patch revision number of the application version.
- **app_version_is_major_release** (BOOLEAN): Flag indicating whether the version was a major release.
- **channel** (STRING): Normalized distribution channel (release, beta, nightly, etc.).

### Operating System Dimensions
- **os** (STRING): Normalized name of the operating system running on the client.
- **os_grouped** (STRING): Operating system group for high-level OS categorization.
- **os_version** (STRING): Complete OS version reported by the client.
- **os_version_major** (INTEGER): Major component of the OS version (e.g., 100 for "100.9.11").
- **os_version_minor** (INTEGER): Minor component of the OS version (e.g., 9 for "100.9.11").
- **os_version_build** (STRING): OS build identifier reported by the client.
- **windows_build_number** (INTEGER): Windows-specific build number (e.g., 22000). NULL for non-Windows platforms.

### Localization & Configuration
- **locale** (STRING): Language and country-based preferences for the user interface.
- **is_default_browser** (BOOLEAN): Flag indicating whether Firefox is set as the default browser on the client.
- **distribution_id** (STRING): Distribution identifier associated with the Firefox installation.

### User Behavior & Attribution
- **activity_segment** (STRING): Classification of users based on browsing activity patterns (e.g., infrequent, casual, regular, heavy).
- **attribution_medium** (STRING): Marketing medium attribution (e.g., referral, organic, paid).
- **attribution_source** (STRING): Marketing source attribution (e.g., google, bing, direct).

### Aggregated Metrics

#### Ping-Based Counts (Submission Activity)
- **daily_users** (INTEGER): Count of users who submitted a ping on the submission_date. Aggregated from `is_daily_user` flag.
- **weekly_users** (INTEGER): Count of users who submitted a ping within the past 7 days from submission_date. Aggregated from `is_weekly_user` flag.
- **monthly_users** (INTEGER): Count of users who submitted a ping within the past 28 days from submission_date. Aggregated from `is_monthly_user` flag.

#### Activity-Based Counts (Browsing Activity)
- **dau** (INTEGER): Daily Active Users - count of users with actual browsing activity on submission_date. Aggregated from `is_dau` flag.
- **wau** (INTEGER): Weekly Active Users - count of users with browsing activity within the past 7 days. Aggregated from `is_wau` flag.
- **mau** (INTEGER): Monthly Active Users - count of users with browsing activity within the past 28 days. Aggregated from `is_mau` flag.

## Update Schedule

- **Frequency**: Daily incremental updates
- **DAG**: `bqetl_analytics_aggregations`
- **Partition**: Day-level partitioning on `submission_date`
- **Partition Filter**: Required for all queries
- **Data Freshness**: Typically updated within 24 hours of the submission date

## Query Performance

The table is optimized for efficient querying with:
- **Partitioning**: By `submission_date` (day-level) with required partition filter
- **Clustering**: By `app_name`, `channel`, and `country`

### Best Practices

1. **Always include a partition filter** on `submission_date`:
   ```sql
   WHERE submission_date BETWEEN '2024-01-01' AND '2024-01-31'
   ```

2. **Leverage clustering columns** in WHERE clauses for better performance:
   ```sql
   WHERE submission_date = '2024-01-15'
     AND app_name = 'Firefox Desktop'
     AND channel = 'release'
     AND country = 'US'
   ```

3. **Choose the appropriate metric** based on your analysis needs:
   - Use `dau`, `wau`, `mau` for engagement analysis (actual browsing activity)
   - Use `daily_users`, `weekly_users`, `monthly_users` for ping submission patterns

## Usage Examples

### Example 1: Daily Active Users by Country
```sql
SELECT
  submission_date,
  country,
  SUM(dau) AS total_dau
FROM
  `moz-fx-data-shared-prod.firefox_desktop_derived.baseline_active_users_aggregates_v2`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
  AND channel = 'release'
GROUP BY
  submission_date,
  country
ORDER BY
  submission_date,
  total_dau DESC
```

### Example 2: Version Adoption Analysis
```sql
SELECT
  submission_date,
  app_version_major,
  SUM(daily_users) AS users,
  SUM(dau) AS active_users
FROM
  `moz-fx-data-shared-prod.firefox_desktop_derived.baseline_active_users_aggregates_v2`
WHERE
  submission_date = '2024-01-15'
  AND channel = 'release'
GROUP BY
  submission_date,
  app_version_major
ORDER BY
  app_version_major DESC
```

### Example 3: Default Browser Penetration
```sql
SELECT
  submission_date,
  country,
  SUM(CASE WHEN is_default_browser THEN dau ELSE 0 END) AS default_browser_dau,
  SUM(dau) AS total_dau,
  SAFE_DIVIDE(
    SUM(CASE WHEN is_default_browser THEN dau ELSE 0 END),
    SUM(dau)
  ) AS default_browser_rate
FROM
  `moz-fx-data-shared-prod.firefox_desktop_derived.baseline_active_users_aggregates_v2`
WHERE
  submission_date BETWEEN '2024-01-01' AND '2024-01-31'
  AND channel = 'release'
GROUP BY
  submission_date,
  country
```

### Example 4: Activity Segment Distribution
```sql
SELECT
  activity_segment,
  SUM(mau) AS monthly_active_users
FROM
  `moz-fx-data-shared-prod.firefox_desktop_derived.baseline_active_users_aggregates_v2`
WHERE
  submission_date = '2024-01-31'
  AND channel = 'release'
  AND country = 'US'
GROUP BY
  activity_segment
ORDER BY
  monthly_active_users DESC
```

### Example 5: Attribution Performance
```sql
SELECT
  attribution_medium,
  attribution_source,
  SUM(daily_users) AS new_daily_users,
  SUM(dau) AS active_users
FROM
  `moz-fx-data-shared-prod.firefox_desktop_derived.baseline_active_users_aggregates_v2`
WHERE
  submission_date = '2024-01-15'
  AND first_seen_year = 2024
GROUP BY
  attribution_medium,
  attribution_source
ORDER BY
  new_daily_users DESC
```

## Related Tables

- **`firefox_desktop.baseline_active_users`**: Source table containing user-level activity flags before aggregation
- **`firefox_desktop.baseline`**: Raw baseline ping data
- **`firefox_desktop_derived.clients_daily_v6`**: Alternative daily aggregation focusing on client-level metrics
- **`telemetry_derived.clients_daily_v6`**: Legacy daily aggregation from main ping

## Data Quality Notes

1. **Unknown Geography**: Country values of '??' indicate unknown or unresolvable IP geolocation
2. **Null Values**: Many dimensional fields can be NULL when the client doesn't report those values
3. **User vs Active User Distinction**: 
   - `daily_users`, `weekly_users`, `monthly_users` count ping submissions
   - `dau`, `wau`, `mau` count actual browsing activity
   - DAU/WAU/MAU are typically lower as they require active engagement
4. **Windows-Specific Fields**: `windows_build_number` is only populated for Windows clients
5. **Version Fields**: Patch revision and release flags may be NULL for older versions

## Owners

- **Primary**: kwindau@mozilla.com
- **Secondary**: ago@mozilla.com

## References

For more information about baseline pings and active user definitions, see:
- [Firefox Telemetry Documentation](https://firefox-source-docs.mozilla.org/toolkit/components/telemetry/)
- [Baseline Ping Schema](https://docs.telemetry.mozilla.org/datasets/pings.html#baseline-ping)

---

**Last Updated**: 2025-01-16  
**Table Type**: Aggregate (Incremental)
