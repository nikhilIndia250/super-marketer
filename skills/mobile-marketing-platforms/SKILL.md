---
name: mobile-marketing-platforms
description: Use when working with mobile attribution via AppsFlyer, Branch, or Adjust - install attribution, attribution windows, deep links, SKAdNetwork, or reconciling install data across sources.
---

# Mobile Marketing Platforms — AppsFlyer, Branch, Adjust

## Mobile Attribution — Core Concepts
- **Attribution** — determining which ad/source caused an app install or in-app event
- **Last touch attribution** — credit given to the last ad the user interacted with before converting
- **View-through attribution** — credit for ad impressions (not clicks) within a window
- **Attribution window** — time after ad interaction during which a conversion is credited (e.g. 7-day click, 1-day view)
- **Re-attribution** — crediting a source for re-engaging a lapsed user (installs the app again)
- **Re-engagement** — crediting a source for bringing back an existing user (deeplink)
- **Fingerprinting** — probabilistic attribution using device signals when deterministic ID not available (being phased out due to privacy)
- **SKAdNetwork (SKAN)** — Apple's privacy-preserving attribution for iOS. Aggregated, no user-level data, delayed reporting.

## AppsFlyer

### Authentication
- API v2: Bearer token (API key from dashboard → API Management)
- Pull API endpoint: `https://hq1.appsflyer.com/api/`
- Master API for MCC-style access across apps

### Key Reports (Pull API)
```
GET /export/{app_id}/installs_report/v5
GET /export/{app_id}/in_app_events_report/v5
GET /export/{app_id}/uninstall_report/v5
GET /export/{app_id}/cohort-report/api/v1    # cohort analysis
GET /export/{app_id}/reach-and-frequency/v1
```

### Key Parameters
```
api_token=...
from=YYYY-MM-DD
to=YYYY-MM-DD
timezone=Asia/Kolkata          # always specify
currency=preferred             # account currency
reattr=true                    # include re-attribution
media_source=facebook,google   # filter by source (optional)
```

### Key Fields — Installs Report
```
Install Time, Event Time, Event Name, Event Value, Event Revenue
Media Source (channel), Campaign, Adset, Ad
Country Code, Device Type, OS Version
IDFA (iOS), GAID (Android), AppsFlyer ID
Attribution Type (Organic, Non-organic, Re-attribution)
Is Retargeting, Retargeting Conversion Type
```

### Key Fields — In-App Events Report
```
Event Name         # af_purchase, af_add_to_cart, af_level_achieved, etc.
Event Value        # JSON blob with event parameters
Event Revenue      # extracted revenue value
Event Currency
Media Source, Campaign, Adset
Attribution Type
```

### Standard AppsFlyer Events (af_ prefix)
```
af_purchase          # purchase completed
af_add_to_cart       # add to cart
af_add_to_wishlist
af_initiated_checkout
af_content_view      # product/content viewed
af_search
af_login
af_complete_registration
af_level_achieved    # gaming
af_tutorial_completion
af_subscribe
af_start_trial
```

### Cohort API
- Analyse user retention and LTV by cohort (install date)
- Metrics: retention rate, revenue per user, events per user
- Grouping: by day/week/month since install

### Aggregate Report API (no raw data)
```
GET /api/agg-data/export/app/{app_id}/partners_by_date/v5
```
Returns aggregated metrics by media source + date — good for dashboard summaries.

### Rate Limits
- Raw data (user-level): 200 requests/day per app
- Aggregate data: higher limits
- Use aggregate where possible, raw data sparingly

---

## Branch

### Authentication
- Branch Key + Branch Secret (from dashboard → Account Settings)
- API endpoint: `https://api2.branch.io/`

### Key Endpoints
```
POST /v1/export           # custom data export
GET  /v1/dashboard/...    # aggregate dashboard data
```

### Export API Request
```javascript
POST https://api2.branch.io/v1/export
{
  "branch_key": "...",
  "branch_secret": "...",
  "export_date": "2026-06-01",   // YYYY-MM-DD, one day at a time
  "response_format": "json"       // or "csv"
}
// Returns download URL — poll until ready, then download
```

### Key Fields
```
id, timestamp, name (event name)
origin                  # BRANCH (deeplink), NATIVE (direct), CROSS_PLATFORM
last_attributed_touch_type   # CLICK, IMPRESSION, WEB_TO_APP_AUTO_REDIRECT
last_attributed_touch_data_tilde_feature  # ads, email, organic, etc.
last_attributed_touch_data_tilde_campaign
last_attributed_touch_data_tilde_channel
days_from_last_attributed_touch_to_event
user_data_os, user_data_platform
user_data_country
deep_linked             # true/false
```

### Standard Events
```
INSTALL, OPEN, WEB_TO_APP_AUTO_REDIRECT
PURCHASE, ADD_TO_CART, VIEW_ITEM, INITIATE_PURCHASE
COMPLETE_REGISTRATION, LOGIN, SUBSCRIBE
ACHIEVE_LEVEL, UNLOCK_ACHIEVEMENT (gaming)
CLICK (ad click through Branch)
```

### Branch Links & Deep Links
- Every Branch link is trackable
- Deep links open to specific app content
- Universal Links (iOS) / App Links (Android)
- Deferred deep linking — user installs app, then lands on correct content

---

## Adjust

### Authentication
- API Token from dashboard
- Base URL: `https://api.adjust.com/`

### Key Endpoints
```
GET /kpis/v1/{app_token}    # KPI report
GET /deliverables/v1/{app_token}  # raw data
```

### KPI Report Parameters
```
start_date=YYYY-MM-DD
end_date=YYYY-MM-DD
kpis=installs,sessions,revenue,daus
dimensions=network,campaign,adgroup,creative
```

---

## Cross-Platform Concepts

### Organic vs Non-organic
- **Organic** — no ad attribution, natural discovery (App Store search, word of mouth)
- **Non-organic** — attributed to an ad/marketing source

### LTV (Lifetime Value) Calculation
- Revenue generated per user over their lifetime
- MMP tracks in-app purchases → calculate LTV per cohort × source

### SKAN (iOS 14+) Challenges
- No user-level data — aggregated only
- Conversion value (0-63) must be configured to represent business value
- Postback delay: 24-48h minimum, often 24-144h
- No campaign-level breakdown in basic SKAN — need SKAN 4.0 for crowd anonymity
- MMPs provide SKAN dashboards alongside regular attribution

### Privacy Frameworks
- **ATT (iOS)** — must request tracking permission. Opt-in rate ~30-40%.
- **Android Privacy Sandbox** — Google's equivalent coming for Android
- **Aggregated Event Measurement (Meta)** — limits to 8 events per domain

### Key Dimensions Across All Platforms
```
Media Source / Channel    # facebook, google_ads, organic, etc.
Campaign Name / ID
Adset / Ad Group Name / ID
Ad Name / Creative ID
Country
Device Type (iOS, Android)
OS Version
App Version
Attribution Type (click, view, re-attribution, re-engagement)
```
