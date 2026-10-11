---
name: google-ads-api
description: Use when writing or debugging code against the Google Ads API - OAuth and developer token auth, MCC login-customer-id headers, GAQL queries, or API error handling.
---

# Google Ads API — Technical Reference

## Authentication
- **OAuth2** — standard flow with refresh tokens
- **Service Account** — for server-to-server (requires domain-wide delegation, not recommended for Ads API)
- **Developer Token** — required in addition to OAuth. Every API call needs: `developer-token` header + OAuth access token
- **Login Customer ID** — MCC account ID in header `login-customer-id` when accessing client accounts via MCC
- Scopes: `https://www.googleapis.com/auth/adwords`

## SDKs
- Node.js: `google-ads-api` (unofficial but widely used) or raw REST
- REST base URL: `https://googleads.googleapis.com/v18/`
- API version: currently v18 (as of 2025). Always pin version in URL.

## Google Ads Query Language (GAQL)
SQL-like language for querying resources:
```sql
SELECT
  campaign.id,
  campaign.name,
  campaign.status,
  metrics.impressions,
  metrics.clicks,
  metrics.cost_micros,
  metrics.conversions,
  metrics.conversions_value
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
  AND campaign.status = 'ENABLED'
ORDER BY metrics.cost_micros DESC
LIMIT 100
```

Key rules:
- `cost_micros` = cost in micros (divide by 1,000,000 to get currency)
- `date` segments use format `YYYY-MM-DD` or `DURING` keywords — but only
  Google's specific predefined literals work (`LAST_7_DAYS`,
  `LAST_14_DAYS`, `LAST_30_DAYS`, `THIS_MONTH`, `LAST_MONTH`, etc.).
  **`LAST_90_DAYS` and other arbitrary "last N days" values are not valid**
  (`INVALID_VALUE_WITH_DURING_OPERATOR`) — for any range outside the fixed
  list, use an explicit `WHERE segments.date BETWEEN 'YYYY-MM-DD' AND
  'YYYY-MM-DD'` instead.
- Cannot mix incompatible resources in one query
- Every query must have a FROM clause (main resource)
- **A field used in `WHERE` must also appear in `SELECT`**, or the query
  is rejected with `EXPECTED_REFERENCED_FIELD_IN_SELECT_CLAUSE` — e.g.
  filtering on `segments.conversion_action_category` requires selecting
  it too, not just referencing it in the WHERE clause.
- GAQL's `WHERE` doesn't support combining multiple `LIKE` conditions with
  `OR` the way plain SQL does — `UNEXPECTED_INPUT` on `OR`. Run separate
  queries per condition instead.

## Schema Introspection (GoogleAdsFieldService)

Query the API's own field/resource metadata via a **different endpoint**
than regular queries: `POST /v{version}/googleAdsFields:search` (not
`/customers/{id}/googleAds:search`) — no customer ID needed, since this
queries the schema itself, not account data.

```sql
SELECT name, category, selectable, data_type, selectable_with, enum_values
WHERE name = 'metrics.conversions'
```

- **This endpoint does not accept a `FROM` clause at all** —
  `UNEXPECTED_FROM_CLAUSE` ("The FROM clause cannot be used in queries to
  any service except GoogleAdsService"). Every other query pattern in
  this file uses `FROM`; this one specifically must not.
- Useful fields on a `google_ads_field` result: `selectable_with` (which
  resources/segments this field can be queried alongside), `enum_values`
  (the real, current enum options for an enum-typed field — don't guess
  these from memory, they change), `category` (`METRIC`/`SEGMENT`/
  `ATTRIBUTE`/`RESOURCE`).

**`selectable_with` is not proof a query actually works — it describes
field-level compatibility, not resource-level query validity.** Confirmed
by two separate live failures where the schema listed a resource as
compatible but the actual query rejected it outright:
- A metric's `selectable_with` listed `ad_group`, but querying `FROM
  ad_group` with that metric failed with `PROHIBITED_METRIC_IN_SELECT_OR_
  WHERE_CLAUSE` naming `AD_GROUP` as unsupported.
- A segment's `selectable_with` implied compatibility with an asset-level
  view resource in general, but combining that segment with that specific
  resource failed with `PROHIBITED_SEGMENT_IN_SELECT_OR_WHERE_CLAUSE`.

**Always live-test a grain/compatibility claim with a real query before
relying on it** — treat `selectable_with` as a hint for what to try next,
never as the final word.

## Resolving opaque ID references (a general pattern, not resource-specific)

Several fields return an opaque resource-name reference instead of a
human-readable value — e.g. `segments.geo_target_state` /
`segments.geo_target_city` return `geoTargetConstants/NNNN`, and
`segments.asset_interaction_target.asset` returns
`customers/{id}/assets/NNNN`. These always need a **second query** against
the referenced resource to resolve:

```sql
-- Resolve a geo target constant
SELECT geo_target_constant.id, geo_target_constant.name,
       geo_target_constant.canonical_name
FROM geo_target_constant WHERE geo_target_constant.id IN (20460, 9061642)

-- Resolve an asset (field to select depends on asset.type)
SELECT asset.id, asset.type, asset.text_asset.text,
       asset.image_asset.full_size.url
FROM asset WHERE asset.id IN (381603169668, 381502948187)
```

This two-query shape (segment/dimension query → collect the opaque IDs →
one batched lookup query against the referenced resource) is the general
pattern for any field that returns a bare resource name rather than an
inline value — don't assume a single query can return the resolved name
directly.

## Conversion Action Fields

The `conversion_action` resource — how a client's conversion tracking is
actually configured, independent of any specific campaign:

```
conversion_action.id
conversion_action.name
conversion_action.category            # PURCHASE, DOWNLOAD, SIGNUP, SUBSCRIBE_PAID, SUBMIT_LEAD_FORM, etc.
conversion_action.status               # ENABLED, HIDDEN, REMOVED
conversion_action.type                 # e.g. FIREBASE_ANDROID_CUSTOM, FIREBASE_ANDROID_IN_APP_PURCHASE, THIRD_PARTY_APP_ANALYTICS_ANDROID_IN_APP_PURCHASE
conversion_action.primary_for_goal     # whether this action is what bidding actually optimizes toward
conversion_action.include_in_conversions_metric  # whether it rolls into the account's "Conversions" total at all
conversion_action.origin
conversion_action.counting_type
conversion_action.firebase_settings.event_name
conversion_action.firebase_settings.property_id
conversion_action.third_party_app_analytics_settings.event_name
conversion_action.third_party_app_analytics_settings.provider_name   # e.g. "AppsFlyer"
conversion_action.app_id
```

**`status: HIDDEN` does not mean excluded from conversions** — Google
auto-creates a conversion action for most event names an SDK ever
reports, and most sit `HIDDEN` (hidden from the UI picker), but a
`HIDDEN` action can still have `include_in_conversions_metric: true` and
count toward the account's totals. Check `include_in_conversions_metric`
and `primary_for_goal` directly — never infer either from `status` alone.

**A real, recurring failure mode worth checking for on any account**: more
than one conversion action simultaneously `ENABLED`,
`primary_for_goal: true`, and `include_in_conversions_metric: true` for
the same underlying business event (e.g. one Firebase-sourced, one
MMP-sourced action both tracking "purchase") — this silently double-counts
in any unsegmented `metrics.conversions` pull. Query `conversion_action`
directly and check for duplicates before trusting an aggregate conversions
number.

## Change History (`change_event`)

The only resource that tells you **who changed what, and when** — the
real way to confirm whether a performance shift was caused by a human
edit (budget, bid target, status, targeting) rather than a platform/
delivery effect. Confirmed live (2026-09-20) diagnosing a real
budget-cut-driven decline — see `google-ads-uac`'s failure-pattern list
for the worked example.

```sql
SELECT change_event.change_date_time, change_event.change_resource_type,
  change_event.user_email, change_event.change_resource_name,
  change_event.changed_fields, change_event.resource_change_operation,
  change_event.old_resource, change_event.new_resource
FROM change_event
WHERE change_event.change_date_time DURING LAST_30_DAYS
  AND change_event.campaign IN ('customers/{id}/campaigns/{campaignId}')
ORDER BY change_event.change_date_time ASC
```

Three real gotchas, all confirmed live, none guessable from the field
names:

- **The resource-type field is `change_event.change_resource_type`, not
  `change_event.resource_type`** — the latter doesn't exist
  (`UNRECOGNIZED_FIELD`). Easy to guess wrong since every other filterable
  "what kind of thing is this" field elsewhere in the API omits the
  `change_` prefix.
- **`change_event.changed_fields` cannot be filtered with `=`** in
  `WHERE` (`OPERATOR_FIELD_MISMATCH` — "Valid operators: ." i.e. none via
  a comparison operator). It's a read-only descriptive field (a
  comma-separated list of which sub-fields changed, e.g.
  `"targetCpa.targetCpaMicros"` or `"amountMicros"`) — filter for a
  specific kind of change client-side after fetching, not in GAQL.
- **`DURING LAST_30_DAYS` can 400 with `START_DATE_TOO_OLD`** right at the
  boundary — the account's current day rolling forward mid-session can
  push a previously-valid 30-day window one day too old before the next
  call. Prefer an explicit `WHERE change_event.change_date_time BETWEEN
  'YYYY-MM-DD' AND 'YYYY-MM-DD'` for anything long-running, same as the
  general `DURING`-vs-`BETWEEN` guidance above.
- Scope to one or more campaigns with `change_event.campaign IN
  ('customers/{id}/campaigns/{campaignId}', ...)` — this also implicitly
  scopes budget changes, since a `CAMPAIGN_BUDGET` change event still
  carries the owning campaign's resource name in this field even though
  the `change_resource_name` itself is the budget's resource name, not
  the campaign's.
- `old_resource`/`new_resource` are shaped exactly like the resource
  itself (e.g. `{"campaignBudget":{"amountMicros":"..."}}` or
  `{"campaign":{"targetCpa":{"targetCpaMicros":"..."}}}`) — diff the two
  directly rather than trying to parse `changed_fields` into a value.
- `change_event.user_email` and `change_event.client_type` (e.g.
  `GOOGLE_ADS_WEB_CLIENT`) tell you whether a change was a manual UI edit
  by a specific person versus an API/automated change — useful for
  distinguishing "someone on the team changed this" from "Smart Bidding
  adjusted itself" (the latter doesn't show up in `change_event` at all —
  automated bid adjustments within a bidding strategy aren't logged here,
  only explicit setting/targeting/status/budget changes are).

## Main Resources
- `campaign` — campaign level data
- `ad_group` — ad group level
- `ad_group_ad` — individual ads
- `keyword_view` — keyword performance
- `search_term_view` — search terms that triggered ads
- `geographic_view` — performance by location
- `audience_view` — audience performance
- `campaign_audience_view` — campaign-level audience
- `asset_group` — Pmax asset groups
- `shopping_performance_view` — shopping campaign data

## Key Metrics Fields
```
metrics.impressions
metrics.clicks
metrics.cost_micros          # divide by 1000000
metrics.ctr
metrics.average_cpc
metrics.conversions
metrics.conversions_value
metrics.all_conversions
metrics.view_through_conversions
metrics.cost_per_conversion
metrics.roas                 # conversions_value / cost
metrics.search_impression_share
metrics.search_top_impression_share
```

## Key Segment Fields
```
segments.date               # YYYY-MM-DD
segments.day_of_week        # MONDAY, TUESDAY, etc.
segments.device             # MOBILE, DESKTOP, TABLET
segments.network            # SEARCH, SEARCH_PARTNERS, CONTENT (display)
segments.ad_network_type
segments.conversion_action  # specific conversion action
```

## Campaign Fields
```
campaign.id
campaign.name
campaign.status             # ENABLED, PAUSED, REMOVED
campaign.advertising_channel_type  # SEARCH, DISPLAY, SHOPPING, VIDEO, PERFORMANCE_MAX, APP
campaign.campaign_budget
campaign.bidding_strategy_type
campaign.start_date
campaign.end_date
campaign.labels
```

## Ad Strength

`ad_group_ad.ad_strength` — an enum rating on the individual ad, **not**
`asset_group.ad_strength` (that's the separate Performance Max/Demand Gen
field; querying `asset_group` on an account with no Pmax/Demand Gen
campaigns returns zero rows). Applies broadly across ad types that build
from multiple assets (Responsive Search Ads, App Campaign ads), not one
specific campaign type.

```sql
SELECT campaign.name, ad_group.name, ad_group_ad.ad.id,
       ad_group_ad.ad_strength, ad_group_ad.status
FROM ad_group_ad
WHERE ad_group_ad.status = 'ENABLED'
```

Enum values, low to high: `INCOMPLETE` (minimum asset requirements not
met — fewer than 2 headlines / 1 description), `POOR`, `AVERAGE`, `GOOD`,
`EXCELLENT`. A row can also come back with the field simply absent/unset
(no rating assigned yet) — don't conflate that with `POOR`; tally it
separately when computing a distribution.

## Rate Limits
- 1,000 operations per request (batch)
- No hard daily limit but excessive use triggers throttling
- Use `page_size` parameter for pagination (default 10,000, max 10,000)
- Use `next_page_token` for pagination

## Making API Calls (Node.js pattern)
```javascript
const { GoogleAdsApi } = require('google-ads-api');

const client = new GoogleAdsApi({
  client_id: process.env.GOOGLE_ADS_CLIENT_ID,
  client_secret: process.env.GOOGLE_ADS_CLIENT_SECRET,
  developer_token: process.env.GOOGLE_ADS_DEVELOPER_TOKEN,
});

const customer = client.Customer({
  customer_id: customerId,
  refresh_token: refreshToken,
  login_customer_id: mccId, // if using MCC
});

const response = await customer.query(`
  SELECT campaign.id, campaign.name, metrics.cost_micros
  FROM campaign
  WHERE segments.date DURING LAST_7_DAYS
`);
```

## Incremental Sync Pattern
- Query by `segments.date` with specific date range
- Store `last_synced_date` in DB
- On each sync: query from `last_synced_date` to yesterday
- Always re-fetch yesterday (data can update up to 3 days after)
- Use `UPSERT` (not INSERT) to handle reprocessed data

## Common Gotchas
- Cost is always in micros — always divide by 1,000,000
- Removed campaigns/ad groups still appear in queries — filter by status if needed
- Pmax campaigns have no keyword_view data — use asset_group resource instead
- Conversion data can be delayed up to 72 hours
- `all_conversions` includes view-through; `conversions` is click-based only
- Date ranges are in the account's local timezone, not UTC
- API version deprecation: Google deprecates old versions — always check current version
