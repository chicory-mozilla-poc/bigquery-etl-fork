# Fenix Addons by Client

## Overview

The `fenix_addons_by_client_v1` table aggregates addon data from Fenix (Firefox for Android) clients across multiple channels. This table combines metrics from Release, Beta, Nightly, and Preview Nightly versions to provide a comprehensive view of addon usage per client per day.

## Data Sources

This table unions data from the following sources:
- **Release**: `org_mozilla_firefox.metrics`
- **Beta**: `org_mozilla_firefox_beta.metrics`
- **Nightly**: `org_mozilla_fenix.metrics`
- **Preview Nightly**: `org_mozilla_fenix_nightly.metrics`
- **Legacy Nightly**: `org_mozilla_fennec_aurora.metrics`

All sources are from the `moz-fx-data-shared-prod` dataset.

## Key Columns

### Client Identifiers
- **client_id**: Unique identifier for each Fenix client
- **sample_id**: Sample identifier used for data sampling
- **submission_date**: Date when the metrics data was submitted

### Application Metadata
- **app_version**: The version of the Fenix application
  - Release/Beta channels use `app_display_version`
  - Nightly channels use `geckoview_version`
  - Beta versions are formatted as `80.0.0b1` (replacing `-beta.` with `b`)

- **country**: Normalized country code based on the most recent value
- **locale**: Client's locale setting
- **app_os**: Normalized operating system

### Addon Data
- **addons**: An array of enabled addons with the following structure:
  - `addon`: Trimmed addon identifier
  - `version`: Addon version (currently NULL as metrics ping doesn't include this data)

## Update Frequency

This table is designed to be updated daily, processing data for each `@submission_date` parameter.

## Important Notes

1. **Version Validation**: Only app versions matching the pattern `^\d+\.\d+(\.\d+)?([ab]\d+)?$` are included (e.g., 80.0, 80.0.0, 80.0.0a1, 80.0.0b1)

2. **Addon Versions**: As of July 1, 2020, the Fenix metrics ping does not contain addon version information, so the `version` field in the addons array is NULL

3. **Aggregation**: Data is aggregated per client per day, with addon lists concatenated across all submissions and other fields using the mode_last function to select the most common value

4. **Null Clients**: Records with NULL client_id are filtered out

## Example Use Cases

1. **Addon Popularity Analysis**: Identify the most commonly installed addons across the Fenix user base by counting distinct clients with each addon

2. **Channel Comparison**: Compare addon adoption rates across different Fenix channels (Release, Beta, Nightly) to understand early adopter behavior

3. **Geographic Distribution**: Analyze addon usage patterns by country to identify regional preferences or trends

4. **Version Correlation**: Examine how addon usage correlates with specific app versions to understand compatibility issues or feature adoption
