# Raw Recipient (Acoustic Campaign Data)

## Table Overview

**Dataset**: `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`

This table contains raw recipient-level email engagement data exported from Acoustic Campaign. It captures detailed metrics around email performance including sends, opens, clicks, bounces, and other recipient interactions. The data is imported from CSV exports provided by Acoustic Campaign's API and serves as the foundation for email marketing analytics.

## Data Source

Data is imported from Acoustic Campaign via their [Raw Recipient Data Export API](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport).

## Table Structure

- **Partitioning**: Day-based partitioning on `submission_date` field
- **Clustering**: Optimized for queries filtering by `event_type`, `recipient_type`, and `body_type`
- **Retention**: 775 days (approximately 2 years)
- **Update Frequency**: Incremental daily loads

## Key Columns

### Identifiers
- `recipient_id`: Contact identifier
- `mailing_id`: Specific email send identifier
- `campaign_id`: Campaign grouping identifier
- `content_id`: Content variant identifier

### Event Data
- `event_timestamp`: When the event occurred
- `event_type`: Type of interaction (Sent, Open, Click, Bounce, etc.)
- `recipient_type`: Email recipient classification
- `body_type`: Email format (HTML, Plain Text)

### Engagement Tracking
- `click_name`: Named link tracking
- `url`: Full clickthrough URL
- `suppression_reason`: Exclusion explanation

## Downstream Analysis Suggestions

### 1. Email Campaign Performance Dashboard
Analyze campaign effectiveness by calculating key metrics:
- **Open rates** by campaign and content variant
- **Click-through rates** by link name and URL
- **Bounce and suppression analysis** to identify deliverability issues
- **Temporal patterns** in engagement by time of day and day of week

```sql
SELECT
  campaign_id,
  event_type,
  COUNT(DISTINCT recipient_id) as unique_recipients,
  COUNT(*) as total_events,
  DATE(event_timestamp) as event_date
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY campaign_id, event_type, event_date
ORDER BY event_date DESC, total_events DESC
```

### 2. Recipient Engagement Segmentation
Build recipient personas based on engagement patterns:
- **High-value recipients**: Multiple clicks and opens
- **At-risk recipients**: No recent engagement
- **Content preferences**: HTML vs Plain Text performance
- **Link engagement analysis**: Most popular URLs and click names

```sql
SELECT
  recipient_id,
  COUNT(DISTINCT CASE WHEN event_type = 'Open' THEN mailing_id END) as opens,
  COUNT(DISTINCT CASE WHEN event_type = 'Click' THEN mailing_id END) as clicks,
  COUNT(DISTINCT mailing_id) as emails_received,
  MAX(event_timestamp) as last_engagement
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY recipient_id
HAVING emails_received > 5
```

### 3. Content Optimization Analysis
Identify which content variants and formats drive better engagement:
- **A/B test results** by content_id
- **Body type performance** (HTML vs Plain Text)
- **Subject line impact** (when joined with campaign metadata)
- **Time-to-open and time-to-click** analysis

```sql
SELECT
  content_id,
  body_type,
  COUNT(DISTINCT recipient_id) as recipients,
  COUNTIF(event_type = 'Open') as opens,
  COUNTIF(event_type = 'Click') as clicks,
  SAFE_DIVIDE(COUNTIF(event_type = 'Open'), COUNT(DISTINCT recipient_id)) as open_rate,
  SAFE_DIVIDE(COUNTIF(event_type = 'Click'), COUNTIF(event_type = 'Open')) as ctr
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND event_type IN ('Sent', 'Open', 'Click')
GROUP BY content_id, body_type
ORDER BY recipients DESC
```

### 4. Deliverability and Suppression Monitoring
Track email deliverability issues and suppression patterns:
- **Bounce rate trends** over time
- **Suppression reason distribution** to identify systemic issues
- **Recipient type analysis** to optimize targeting
- **Campaign health scores** based on deliverability metrics

```sql
SELECT
  submission_date,
  suppression_reason,
  event_type,
  COUNT(DISTINCT recipient_id) as affected_recipients,
  COUNT(*) as total_events
FROM `moz-fx-data-shared-prod.acoustic_derived.raw_recipient_v1`
WHERE suppression_reason IS NOT NULL
  AND submission_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY submission_date, suppression_reason, event_type
ORDER BY submission_date DESC, total_events DESC
```

## Owners

- **Primary Owner**: cbeck@mozilla.com

## Additional Resources

- [Acoustic Campaign API Documentation](https://developer.goacoustic.com/acoustic-campaign/reference/rawrecipientdataexport)
- Dataset Metadata: `acoustic_derived/dataset_metadata.yaml`

---

*Last Updated: 2025*
