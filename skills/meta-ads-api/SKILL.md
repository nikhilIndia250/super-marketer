---
name: meta-ads-api
description: Use when writing or debugging code against the Meta Marketing API - Graph API auth, system user tokens, insights endpoints, rate limits, or API error handling.
---

# Meta Ads API — Technical Reference

## Authentication
- **OAuth2** — user grants permission via Facebook Login
- App ID + App Secret required (from Meta for Developers)
- Scopes needed: `ads_read`, `ads_management`, `read_insights`, `business_management`
- Access tokens: User token → System User token (recommended for production)
- **System User tokens** — non-expiring, tied to a Business Manager system user. Best for server-to-server.
- Long-lived tokens: exchange short-lived (1h) for long-lived (60 days) using app secret

## Base URL & Versioning
```
https://graph.facebook.com/v21.0/
```
Always pin to a specific version. Meta deprecates old versions after ~2 years.

## Key Endpoints

### Get Ad Accounts
```
GET /me/adaccounts?fields=id,name,currency,account_status
```

### Get Campaigns
```
GET /act_{ad_account_id}/campaigns
  ?fields=id,name,status,objective,daily_budget,lifetime_budget,created_time
  &limit=100
```

### Get Insights (core reporting endpoint)
```
GET /act_{ad_account_id}/insights
  ?fields=campaign_id,campaign_name,adset_id,adset_name,ad_id,ad_name,
          impressions,clicks,spend,reach,frequency,
          actions,action_values,cost_per_action_type,
          ctr,cpc,cpm,cpp,
          video_play_actions,video_p25_watched_actions,video_p50_watched_actions,
          video_p75_watched_actions,video_p100_watched_actions
  &time_range={"since":"2026-01-01","until":"2026-01-31"}
  &level=ad              # campaign | adset | ad
  &breakdowns=age,gender # optional breakdowns
  &time_increment=1      # 1 = daily breakdown, "monthly", "all_days"
  &limit=500
  &after={cursor}        # pagination cursor
```

## Actions Field (conversions)
Actions are returned as an array — must find by action_type:
```javascript
function getActionValue(actions, actionType) {
  if (!actions) return 0;
  const action = actions.find(a => a.action_type === actionType);
  return action ? parseFloat(action.value) : 0;
}

// Common action types:
// 'purchase'               — purchases
// 'add_to_cart'            — add to cart
// 'initiate_checkout'      — checkout started
// 'lead'                   — leads
// 'app_install'            — app installs
// 'mobile_app_install'     — mobile app installs
// 'offsite_conversion.fb_pixel_purchase' — pixel purchases
// 'link_click'             — link clicks
// 'video_view'             — video views (3s)
// 'omni_purchase'          — cross-channel purchase
```

## Breakdowns (dimensions for segmentation)
```
age                    # 13-17, 18-24, 25-34, 35-44, 45-54, 55-64, 65+
gender                 # male, female, unknown
country
region
dma                    # US Designated Market Area
publisher_platform     # facebook, instagram, audience_network, messenger
platform_position      # feed, story, reels, video_feeds, etc.
device_platform        # mobile, desktop
impression_device      # iPhone, Android phone, iPad, etc.
```
Note: Some breakdowns cannot be combined (API returns error). Test combinations.

## Async Reporting (for large date ranges)
For large queries use async report jobs:
```javascript
// 1. Create report job
POST /act_{id}/insights
body: { level: 'ad', time_range: {...}, fields: [...] }
// Returns: { report_run_id: "123" }

// 2. Poll until complete
GET /{report_run_id}
// async_status: "Job Not Started" | "Job Started" | "Job Running" | "Job Completed"

// 3. Fetch results
GET /{report_run_id}/insights?limit=500
```

## Rate Limits
- **Score-based system** — each app has a usage score. Heavy use increases score, pausing lets it recover.
- Score > 100% = throttled (429 errors)
- Use `x-business-use-case-usage` or `x-app-usage` response headers to monitor usage
- Recommended: max 1 request/second for insights, batch where possible
- Batch API: up to 50 requests in one HTTP call

```javascript
// Batch example
POST /
body: {
  batch: [
    { method: 'GET', relative_url: 'act_123/insights?...' },
    { method: 'GET', relative_url: 'act_456/insights?...' },
  ],
  access_token: '...'
}
```

## Pagination
```javascript
// Response includes paging cursors
{
  data: [...],
  paging: {
    cursors: { before: "...", after: "..." },
    next: "https://graph.facebook.com/..."  // only present if more pages
  }
}

// Follow `next` URL until it's absent
```

## Node.js Pattern
```javascript
const axios = require('axios');

const BASE = 'https://graph.facebook.com/v21.0';

async function getInsights(adAccountId, accessToken, dateRange) {
  const params = {
    access_token: accessToken,
    fields: 'campaign_name,adset_name,impressions,clicks,spend,actions,action_values',
    time_range: JSON.stringify(dateRange),
    level: 'adset',
    time_increment: 1,
    limit: 500,
  };

  let results = [];
  let url = `${BASE}/act_${adAccountId}/insights`;

  while (url) {
    const res = await axios.get(url, { params });
    results = results.concat(res.data.data);
    url = res.data.paging?.next || null;
    params = {}; // next URL includes all params
  }

  return results;
}
```

## Incremental Sync Pattern
- Use `time_range: { since: lastSyncDate, until: yesterday }`
- Always re-fetch last 3 days (Meta attribution can update retroactively)
- Store `ad_account_id + date + adset_id` as composite key for upsert
- Attribution window setting must match what you stored previously

## Common Gotchas
- `spend` is returned as a string — always parseFloat()
- `actions` is an array — must find by action_type, never index directly
- iOS14 limits signal for app campaigns — use SKAN data from AppsFlyer/Branch alongside Meta for full picture
- Deleted campaigns/adsets still appear in historical insights — this is correct
- `effective_status` vs `status` — effective_status shows actual delivery status (campaign paused can pause adsets even if adset is ACTIVE)
- Time zone: insights are reported in the ad account's timezone — normalise to UTC when storing
- Budget fields are in the account currency's smallest unit (cents for USD, paise for INR)
