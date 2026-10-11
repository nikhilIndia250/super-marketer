---
name: web-analytics
description: Use when working with GA4 - the event-based data model, sessions, conversions, the reporting API, or reconciling GA4 numbers against ad platform reporting.
---

# Web Analytics — GA4 & Platform Reference

## GA4 Core Concepts

### Data Model (event-based, not session-based like UA)
- Everything is an **event** with **parameters**
- No more pageviews as primary metric — pageview is just an event
- **Sessions** are derived from events (session_start event)
- **Users** identified by Client ID (GA cookie) or User ID (logged-in)

### Key Entities
- **Property** — a GA4 property tied to an app/website
- **Data Stream** — web stream, iOS stream, or Android stream under a property
- **Measurement ID** — G-XXXXXXXXXX (web stream identifier)
- **App Instance ID** — unique per app install (mobile)

### Standard Events (auto-collected)
```
page_view               # every page load
session_start           # start of new session
first_visit             # first time user visits
user_engagement         # user engaged for 10+ seconds
scroll                  # 90% scroll depth
click                   # outbound link clicks
file_download           # file downloads
video_start/progress/complete  # YouTube embedded
```

### Recommended Events (implement manually)
```
purchase                # e-commerce purchase
add_to_cart
remove_from_cart
view_item
view_item_list
begin_checkout
add_payment_info
add_shipping_info
select_item
view_cart
search
sign_up
login
generate_lead
```

### Key Metrics
```
sessions                    # total sessions
engaged_sessions            # sessions lasting 10s+ or 2+ pageviews or conversion
engagement_rate             # engaged_sessions / sessions
average_session_duration
bounce_rate                 # 1 - engagement_rate (inverse of engaged sessions)
active_users / total_users / new_users / returning_users
screen_page_views           # pageviews
views_per_session
event_count
conversions                 # marked conversion events
revenue / total_revenue     # e-commerce
ecommerce_purchases
purchase_revenue
average_purchase_revenue
```

### Key Dimensions
```
date / dateHour / dayOfWeek / week / month / year
country / region / city / continent / subContinent
browser / operatingSystem / deviceCategory (desktop/mobile/tablet)
platform (web/iOS/Android)
sessionSource / sessionMedium / sessionCampaignName (traffic source)
source / medium / campaign / keyword / content
landingPage / pagePath / pageTitle
eventName
customEvent:parameter_name   # custom event parameters
```

## GA4 Data API

### Authentication
- Service Account (recommended for server-to-server)
- OAuth2 (for user-specific access)
- Scope: `https://www.googleapis.com/auth/analytics.readonly`

### Base URL
```
https://analyticsdata.googleapis.com/v1beta/properties/{propertyId}:runReport
```

### Report Request Structure
```javascript
POST https://analyticsdata.googleapis.com/v1beta/properties/{propertyId}:runReport

{
  "dateRanges": [
    { "startDate": "2026-01-01", "endDate": "2026-01-31" }
  ],
  "dimensions": [
    { "name": "date" },
    { "name": "sessionSource" },
    { "name": "sessionMedium" },
    { "name": "deviceCategory" },
    { "name": "country" }
  ],
  "metrics": [
    { "name": "sessions" },
    { "name": "engagedSessions" },
    { "name": "totalUsers" },
    { "name": "newUsers" },
    { "name": "screenPageViews" },
    { "name": "conversions" },
    { "name": "totalRevenue" }
  ],
  "limit": 10000,
  "offset": 0,
  "dimensionFilter": {
    "filter": {
      "fieldName": "sessionSource",
      "stringFilter": { "value": "google" }
    }
  }
}
```

### Response Structure
```javascript
{
  dimensionHeaders: [{ name: "date" }, { name: "sessionSource" }, ...],
  metricHeaders: [{ name: "sessions" }, ...],
  rows: [
    {
      dimensionValues: [{ value: "20260101" }, { value: "google" }, ...],
      metricValues: [{ value: "1234" }, ...]
    }
  ],
  rowCount: 5000,
  metadata: { currencyCode: "INR", timeZone: "Asia/Kolkata" }
}
```

### Node.js SDK Pattern
```javascript
const { BetaAnalyticsDataClient } = require('@google-analytics/data');

const client = new BetaAnalyticsDataClient({
  keyFilename: './service-account.json'  // or credentials object
});

const [response] = await client.runReport({
  property: `properties/${propertyId}`,
  dateRanges: [{ startDate: '30daysAgo', endDate: 'yesterday' }],
  dimensions: [{ name: 'date' }, { name: 'sessionDefaultChannelGroup' }],
  metrics: [{ name: 'sessions' }, { name: 'totalRevenue' }],
});

// Parse response
const rows = response.rows.map(row => {
  const obj = {};
  row.dimensionValues.forEach((v, i) => obj[response.dimensionHeaders[i].name] = v.value);
  row.metricValues.forEach((v, i) => obj[response.metricHeaders[i].name] = parseFloat(v.value));
  return obj;
});
```

### Relative Date Strings (convenience)
```
today, yesterday
NdaysAgo (e.g. 7daysAgo, 30daysAgo)
```

### Realtime API (last 30 minutes)
```
POST .../properties/{id}:runRealtimeReport
```

### Quota Limits
- 200,000 tokens per project per day
- 2,000 tokens per property per hour
- Standard reports: 10 tokens each
- Use daily aggregated pulls, not minute-by-minute

## Channel Groupings (sessionDefaultChannelGroup)
```
Organic Search      # SEO traffic
Paid Search         # Google/Bing ads
Organic Social      # social posts (not ads)
Paid Social         # social ads
Direct             # direct URL or dark social
Referral           # other websites
Email              # email campaigns
Affiliates
Display
Organic Video
Organic Shopping
```

## GA4 vs Universal Analytics (UA) Key Differences
| UA | GA4 |
|---|---|
| Sessions-based | Events-based |
| Bounce rate | Engagement rate |
| Goals | Conversions (any event) |
| Views | Data Streams |
| Pageviews as primary metric | Events as primary |
| Sampling in free tier | No sampling (up to 1M rows) |
| Hit limit 10M/month | Event limit 1000/month per user |

## Common Pitfalls
- `date` dimension returns `YYYYMMDD` format — convert to `YYYY-MM-DD` when storing
- Revenue metrics require e-commerce implementation — may be 0 if not set up
- `totalRevenue` includes tax/shipping — `purchaseRevenue` is net revenue
- Sessions can differ from ad platform sessions (different attribution models)
- GA4 data is not real-time — standard reports have 24-48h latency; use Realtime API for live data
- Property ID ≠ Measurement ID. Property ID is numeric (e.g. 123456789). Use Property ID for API.
- Sampling: free GA4 has no sampling, but BigQuery export is needed for very large custom queries
- Data retention: default 2 months (can set to 14 months in property settings)
