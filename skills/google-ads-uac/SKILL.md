---
name: google-ads-uac
description: Use when diagnosing or optimizing a Google Ads App Campaign (UAC) using this dashboard's own data — what levers actually move an automated bidding campaign, common failure patterns specific to app-install campaigns, and how to correctly read this codebase's attribution numbers for this campaign type (they behave differently from Meta/Search). Campaign-type playbook for performance-marketer; the google-ads skill covers general platform concepts, this one covers diagnosis and this dashboard's own data shape.
---

# Google Ads App Campaigns (UAC) — Diagnostic Playbook

For general Google Ads platform concepts (bidding strategy names, account
structure, campaign types), see the `google-ads` skill. This skill is
specifically about **diagnosing an App Campaign's real performance using
this dashboard's data**, and the one thing about that data you cannot skip.

## Read this first: the attribution caveat is not optional

**Confirmed against real production data this session, not a general
claim**: for every client synced through this dashboard's
`googleAdsFetcher.js`, the ad-level query (`ad_group_ad`) never returns an
ad name for a Google App Campaign — 100% of rows, every client, every
campaign. This is because App Campaigns don't create individually-named
ads the way Search does — there is no manually-built "ad" object for
this campaign type, so the resource this fetcher queries is structurally
empty for it, on every client, always.

**What's actually available, by grain and by data source** (preclick =
Google's own reporting; postclick = installs/in-app actions attributed
by an MMP — AppsFlyer/Adjust/Branch — since App Campaigns don't route
through GA4/web analytics):

| Grain | Preclick: impressions / clicks / cost | Postclick: installs / in-app actions |
|---|---|---|
| Campaign | ✅ Available | ✅ Available |
| Ad Group / Ad Set | ✅ Available | ✅ Available — **the finest grain installs/in-app actions ever reach**, for a platform reason, not a dashboard one |
| Ad | ❌ N/A — no ad object exists for App Campaigns | ❌ N/A — same reason |
| Asset (individual headline/image/video) | ✅ Available from Google via `ad_group_ad_asset_view` — **not currently ingested by this dashboard** | ❌ Not available from any MMP — Google never passes an asset-level identifier to attribution partners for UAC installs |

**The one thing to internalize**: preclick metrics can in principle go
one grain finer (asset) than postclick metrics ever can. Cost/clicks/
impressions could theoretically be seen per-asset if this dashboard
built that fetcher (see "What Google can report that this dashboard
doesn't pull yet" below) — but installs and in-app actions are
permanently capped at ad-group grain, because that ceiling lives in what
Google hands the MMP, not in anything this codebase does or doesn't
query. No future fetcher fixes the installs/in-app-actions row.

This dashboard's `buildGranularRows` already knows this: `nativeDepth()`
caps a Google App Campaign row's matching depth at **adset**, one level
short of where Meta campaigns reach. As of the 2026-09-12 recursive
residual-attribution fix (`docs/DECISIONS.md`), any postclick data that
would have matched at the ad level instead surfaces as an explicit
**adset-level residual row** — so the adset-level and campaign-level
totals are correct and complete, but the **ad-level breakdown for a Google
App Campaign will always be empty or near-empty**, structurally, regardless
of how the campaign is actually performing.

**What this means for you, diagnosing a Google App Campaign:**
- Never conclude "this ad is dead" or "creative X is underperforming" from
  ad-level data on a Google App Campaign — there usually isn't any, by
  design of Google's own API, not because something is broken.
- Do your diagnosis at the **adset** level (Google calls these "ad
  groups," but for App Campaigns there effectively is no ad group either —
  what you're really looking at is asset-group-equivalent granularity via
  this dashboard's `adset` dimension) or the **campaign** level.
- If you see meaningful ad-level data for a client's Google campaigns
  anyway, check whether that client is actually running Search/Display/
  Pmax rather than App Campaigns specifically — this caveat is App
  Campaigns only.
- A residual row (no `adname` on the row, `postclick_tracked: false`) at
  the adset level is expected and correct for this campaign type — it is
  not missing data, it's recovered data. Don't discount it.
- If the client has the Ad-Level Health Indicator enabled (`adHealth` in
  their config — confirmed enabled for firstclub-v2), don't expect it to
  say anything useful for a Google App Campaign. **Confirmed against
  `services/adHealth.js`'s `classifyAd()`**: it needs `postclick_tracked`
  and a per-ad `conversions_attributed` to classify anything; a Google
  App Campaign "ad" row is really the adset-level residual with no ad
  name, so it reliably returns `state: 'insufficient-data'` (reason
  `'no postclick source attributes at this grain'` or `'not attributable
  at this grain'`, depending on the row). A grey/insufficient-data
  verdict here is the expected, correct outcome for this campaign type —
  it means "no ad grain exists to score," not "the feature is broken" or
  "these ads are healthy by default."

**A real, now-fixed bug lived in exactly this adset-capping behavior —
important context if you're reasoning about any firstclub-v2 Google number
from before 2026-10-05.** When a client reports Google rows broken out by
a dimension finer than the capped grain (firstclub-v2 reports `network` —
Search/Content/YouTube — one level past where `nativeDepth()` caps
attribution matching for App Campaigns), every one of those sibling rows
shared the single capped lookup key and **each independently claimed the
full adset-level postclick total** — not a split, a full duplicate claim
per sibling. Confirmed live: `FC_Perf_Google_RnB_App_Android_17072026`
(the clean-attribution reference campaign used throughout this skill) had
three sibling rows each claiming the same 178 `first_transactions`.
Blended across firstclub-v2's whole Google book, this overcounted
`first_transactions`/`conversions_attributed` by **3.6-3.9x**, every day —
a structural join defect, not a one-day glitch, and not visible as an
error anywhere (believable, plausible-looking inflated numbers, same
failure shape as the SuperMoney-V2 `conversions_attributed` bug below).
**Fixed and deployed 2026-10-05** (`docs/DECISIONS.md`, "attribution-sibling
collision" entries, 2026-10-03/10-05) — a single-claim guard
(`attributionClaimLoser`/`groupsByClaimKey`) now ensures exactly one
sibling claims the real postclick figure, chosen deterministically, never
by which one spent the most. The remediation script was run against
firstclub-v2's stored data the same day (33,757 rows rewritten) — verified
directly against production: firstclub-v2's blended 7-day
`first_transactions` total dropped **44,648 → 25,231 (-43.5%)**. Any
pre-2026-10-05 analysis, screenshot, or client conversation referencing
firstclub-v2's Google conversion numbers was working from inflated data —
treat it as stale, not as a baseline to reconcile against. **Not yet
checked for any other client** — any Google Ads client reporting `network`
(or another dimension finer than the postclick source can reach) under a
capped grain is structurally exposed to the same bug; don't assume it's
firstclub-v2-specific without checking that client's own
`reportingDimensions`.

## `conversions_platform` vs `conversions_attributed` — expect a large gap, and know which one to trust

**Confirmed against real data (firstclub-v2, Sept 11, 2026, Google
channel): `conversions_platform` = 6,469 vs `conversions_attributed` =
647 — a ~10x gap.** This is not a data-quality bug on either side; it's
two genuinely different definitions of "conversion":

- `conversions_platform` is Google's own raw "conversions" metric — this
  sums **every conversion action the account has marked "Include in
  conversions"**, which for an app-install account is often installs +
  app opens + multiple in-app engagement events all at once, not one
  specific business outcome.
- `conversions_attributed` is this dashboard's postclick-source-derived
  number (from AppsFlyer/Branch/Adjust/a client sheet) for one specific
  event — usually `first_transactions`, a single business-meaningful
  action.

**For diagnosing real business performance (CAC, whether spend is
working), always use `conversions_attributed`, never
`conversions_platform`.** The platform figure is useful only for
sanity-checking that Google's own tracking is firing at all (a
`conversions_platform` of zero when spend is non-zero is a real
tracking problem — that's a `martech-expert` question) — it is not a
stand-in for business results, and the gap size itself tells you
nothing about performance without knowing which conversion actions are
actually configured on that account.

### `conversions_platform` has its own internal ceiling: "Conversions" vs. "Participated in-app actions"

Even setting aside the gap above, Google's own `metrics.conversions` (this
dashboard's `conversions_platform`) is *itself* the narrower of two figures
Google tracks internally for an ACi (App Campaigns for Installs) campaign —
verified against Google's own documentation, 2026-09-16:

- **Conversions** — post-install in-app actions attributed to *this specific*
  ACi campaign under **last-click** logic only. If the account also runs an
  App Engagement (ACe) or deep-linked campaign, and one of those wins
  last-click credit for a later event, the ACi campaign's "Conversions"
  count excludes that event entirely — even though the ACi campaign is what
  drove the install in the first place.
- **Participated in-app actions** — the broader figure: every post-install
  conversion the ACi campaign contributed to, *including* ones also
  credited to another campaign type via Google's "attribution sharing"
  feature. This exists specifically because App Campaigns historically only
  supported last-click attribution, which under-credits install campaigns
  for their real downstream impact.

**Confirmed against Google's own dedicated documentation
([support.google.com/google-ads/answer/9996103](https://support.google.com/google-ads/answer/9996103?hl=en),
checked 2026-09-20)**: attribution sharing spans exactly four campaign
types — ACI, ACE, ACpre (App Pre-Registration), and App deep-linked
campaigns — and is **automatically enabled by default**, not an opt-in
step (contrary to how the "Enable attribution sharing" best-practice
bullet below used to read before this check — the real action is
confirming it *hasn't* been manually disabled under Settings →
Conversions → App attribution sharing, not turning it on). The same page
also names a sibling metric not previously documented in this skill:
**"Participated Installs"** — the ACI-side counterpart to Participated
in-app actions, covering installs shared between pre-registration and
install campaigns. **Also confirmed API-absent** by the same 2026-09-16
schema search below (it searched for any field matching `participat`
anywhere in the entire schema and found nothing — a hypothetical
`participated_installs`-style field would have matched that search too,
so this isn't a separate untested guess, it's covered by evidence
already gathered) — same UI-only, no-API status as "Participated in-app
actions" below.

**Confirmed live against firstclub-v2's account (2026-09-16): "Participated
in-app actions" is not queryable via the Google Ads API at all.** Searched
the live GAQL field schema (`google_ads_field` via `GoogleAdsFieldService`)
for anything matching `participat`, `attribution_shar`, or `shared` —
nothing related to attribution sharing exists anywhere in the schema; it's
a Google Ads **web UI-only** report metric. That means: `conversions_platform`
as ingested by this dashboard is *always* the narrower, individually-
attributed last-click figure, and there is no API path to the broader
"participated" number — if a client asks why an ACi campaign's numbers here
look lower than what they see in the Google Ads UI's own campaign table,
this is a real, structural reason, not a sync bug. Checking the true
broader figure requires looking directly in the Google Ads UI, outside
this dashboard.

**The equality (or gap) between "In-app actions" and "Participated in-app
actions" is itself a diagnostic tell, not just a definitions curiosity.**
"In-app actions" is the individually-attributed count for one specific
campaign, last-click, no sharing — the narrower sibling of "Conversions."
Confirmed on a real firstclub-v2 campaign export (`FC_Perf_Google_RnB_App_
Android_17072026`, 2026-09-16): In-app actions, Conversions, and Purchase
conversions were all **exactly 3,415.00** — identical. That equality means
no other campaign in the account is currently sharing or claiming any
credit for what this campaign drove. If a client's Google Ads UI export
later shows "Participated" pulling ahead of "In-app actions"/"Conversions"
for the same campaign, that gap opening up is the tell that attribution
sharing has started mattering (an engagement or deep-linked campaign is now
winning last-click on some of this campaign's downstream events) — read it
as a sign the account's campaign mix changed, not as a tracking problem to
chase with `martech-expert`.

## How ACI conversions actually get tracked — four independent mechanisms, any combination can be live at once

This is the reason duplicate/overlapping conversion actions exist at all
(see the AppsFlyer-vs-Firebase finding below) — an account isn't limited
to one tracking pathway, and nothing stops two from being wired up for
the same event simultaneously. Four real, independent mechanisms (iOS/
SKAdNetwork deliberately excluded from this list — it's a modifier on the
MMP path for iOS specifically, not a fifth mechanism):

| Mechanism | How it works | Confidence |
|---|---|---|
| **Google Play Integration** (Android only) | Automatic, zero-code — because Google owns both Ads and Play, an install-through-open can be tracked with no SDK integration at all. **Limited to exactly two conversion types: installs and in-app purchase** — cannot track an arbitrary custom event the way Firebase/MMP can. | Domain knowledge, **not verified against live data** — no current client uses this mechanism, so there's nothing to check it against |
| **Google Analytics for Firebase** | App embeds the Firebase SDK; Firebase logs app opens/events and pings Google Ads to match the user to the ad they saw. Shows up as `conversion_action.firebase_settings.event_name` (e.g. `first_open`, `first_purchase_done`) | **Verified extensively live** — this is the account's own Firebase-sourced tracker in the duplicate-tracking finding below |
| **Third-party MMP** (AppsFlyer / Adjust / Singular / Branch) | MMP generates a tracking link, records the device identifier on click, matches it on app open, sends a postback to Google Ads. Shows up as `conversion_action.third_party_app_analytics_settings.provider_name` | **Verified extensively live** — this account's canonical purchase tracker (AppsFlyer) throughout this whole investigation |
| **Server-to-server via the Google Ads API** | Enterprise pattern — the advertiser's own backend tracks the action and pushes the conversion directly to the Google Ads API, no SDK/MMP in the middle at all | Domain knowledge, **not verified against live data** — no current client uses this mechanism either |

**Before trusting any general knowledge about "how conversions are
configured" — including from `performance-marketer` itself — check it
against this table.** Quizzed cold on "how are conversions configured and
via which tools," the agent's answer (2026-09-19) named only two of these
four (Firebase and third-party MMP) — a real, meaningful completeness gap
for a question posed exactly this way, even though everything it *did*
say was accurate and correctly tied to real `conversion_action` fields.
Google Play Integration and server-to-server API tracking were absent
entirely from its answer.

### tROAS specifically requires Firebase or Google Play Codeless for value — MMP-sourced value has real limitations

**Confirmed** (2026-09-19): for Target ROAS bidding on App Campaigns, Google
requires conversion **value** to flow through the **Google Analytics for
Firebase SDK or Google Play Codeless tracking** specifically. A traditional
third-party MMP integration (AppsFlyer, Adjust) commonly faces real
technical limitations passing value for bidding purposes **unless it's
mapped through Firebase** — an MMP-sourced conversion action isn't simply
an equivalent, interchangeable value source for tROAS the way it is for
tCPA/volume-based bidding.

**This reframes the AppsFlyer-vs-Firebase duplicate-tracking finding
below** — having both live simultaneously isn't necessarily pure
redundancy/risk, the way it reads when you only know about the plain
Conversions-blending problem. If a client wants (or already runs) tROAS,
the Firebase-sourced action isn't a duplicate to eliminate — **it may be
the only one of the two actually eligible to drive value-based bidding at
all**, even while the MMP-sourced action stays the more trustworthy
number for postclick reporting/reconciliation. Don't recommend disabling
either tracker without first checking (a) which bidding strategy the
campaign actually runs, and (b) whether Firebase is the one carrying real
per-event value data if it does run tROAS.

### A second Firebase advantage — audience exclusion segments, not bidding mechanics

**Confirmed against Google's own documentation**
([support.google.com/google-ads/answer/9865058](https://support.google.com/google-ads/answer/9865058?hl=en),
checked 2026-10-10): "Firebase rule-based segments" is one of four
supported Audience Exclusion list types for ACI (alongside Google Play
Audience Segments, Device ID/Customer Match lists, and Google Analytics
audience segments). This is a genuine, still-current Firebase-specific
capability, separate from the tROAS value point above — it lets a client
build custom exclusion audiences from Firebase-tracked in-app behavior
(e.g. excluding repeat-uninstallers or known low-value segments) that
wouldn't exist without Firebase integrated, regardless of what the
campaign actually bids on.

**One nuance worth being precise about — this is not a current exclusive
advantage of *bidding* on Firebase.** The same page states Google removed
a prior requirement: "You no longer need to bid on Firebase to use
Audience Exclusions." So bidding on Firebase used to be a hard
prerequisite just to unlock Audience Exclusions for ACI at all — that
dependency has since been eliminated, and exclusions now work regardless
of bidding configuration. Don't present "use Firebase so you can use
exclusions" as current advice; the real, current advantage is narrower —
having Firebase *integrated* (not necessarily bid on) gives access to
Firebase rule-based segments as one exclusion-list option among several.

## No Google-side metric is universally "the" comparison to postclick conversions — a procedure, not a fixed column

This dashboard's `conversions_platform` (Google's raw, unsegmented
`metrics.conversions`, summed with no category filter) should **never** be
compared directly against `conversions_attributed` — they are not the same
kind of number. Confirmed by fully tracing one real account's numbers
(firstclub-v2, Google channel, 2026-09-12 to 2026-09-18):

| | Value | What it is |
|---|---|---|
| `conversions_platform` (raw, unsegmented) | 42,619 | Every live conversion action summed — installs and purchases together |
| — install-category actions only (`DOWNLOAD`) | 39,485 | "First open," both platforms — **93% of the raw total** |
| — purchase-category actions, all of them | 3,704 | Includes a duplicate tracker (see below) |
| — **purchase-category, single canonical action** | **3,295** | The one number that's a fair comparison |
| `conversions_attributed` / `first_transactions` (postclick) | 5,570 | AppsFlyer, via the Google Sheets connector |

39,485 + 3,704 = 43,189 (the live-API equivalent of the 42,619 above, small
gap from normal conversion-lag revision) — confirms there is no third
category hiding in the total; it is exactly installs plus purchases, mixed
together under one label.

**For ACI specifically, check reinstalls before treating the install
total as new-user growth.** `DOWNLOAD`-category volume, as reported
above, is itself not guaranteed to mean first-ever install — a device
that previously had the app, removed it, and installed it again is a
different business event from a genuinely new user, and ACI's whole
premise (acquiring new users) is undermined if a material share of its
"installs" are actually reinstalls. **Not yet live-tested, flag before
relying on it**: MMPs (Branch/AppsFlyer/Adjust) commonly distinguish
`installs` from a separate `reinstalls` metric at the postclick layer —
check whether the client's postclick source reports one (the same
`postMetricSums` machinery that already carries every metric a
funnel-shaped source reports would surface it, even if it isn't a named
dashboard metric today). Whether Google Ads' own `DOWNLOAD` category
distinguishes first-install from reinstall at the platform layer is
unconfirmed — "First open" terminology suggests it might track
per-session first-open rather than true device-level install history,
which would NOT be the same thing as an MMP's reinstall flag; don't
assume either source's definition without a live check against a real
account. ACE doesn't have this concern structurally — it targets
existing app users by design, so a "reinstall" isn't a meaningful
distinct category there the way it is for ACI's new-user premise.

**Getting from 42,619 to the correct 3,295 requires a real procedure, not a
different column name to swap in:**

1. **Segment by `segments.conversion_action_category` (or
   `conversion_action_name`) and exclude `DOWNLOAD`** (installs). This is
   the single biggest correction — 93% of the raw total on this account was
   installs, not purchases.
2. **Check for duplicate live trackers in the remaining category and pick
   one canonical action.** This account has two simultaneously-live,
   both-`primary_for_goal: true`, both-`include_in_conversions_metric:
   true` purchase actions for the same app — one from AppsFlyer
   (`third_party_app_analytics_settings.provider_name: "AppsFlyer"`), one
   from Firebase SDK directly. Summing the category isn't enough; you have
   to identify which one actually corresponds to the same event the
   postclick source tracks (here, the AppsFlyer one, since the postclick
   source itself is AppsFlyer-via-Google-Sheets) and exclude the rest.
   Query `conversion_action` directly (`include_in_conversions_metric`,
   `primary_for_goal`, `third_party_app_analytics_settings.provider_name`,
   `firebase_settings.event_name`) to find this — don't assume a category
   filter alone is safe.
3. **There is no universal category name to filter to.** "Purchase" only
   applies because this client's postclick event happens to be a purchase.
   A lead-gen or subscription app's canonical category could be
   `SUBMIT_LEAD_FORM`, `SUBSCRIBE_PAID`, `SIGNUP`, or something else
   entirely — the category must match whatever the *specific client's*
   postclick metric actually tracks, determined per client, not assumed.

**Even after doing this correctly, expect a real remaining gap** — 3,295
(corrected preclick) vs. 5,570 (postclick) is still a ~69% difference on
this account, likely an attribution-window mismatch between what's
configured on the Google-side conversion action and AppsFlyer's own
lookback settings. That's a `martech-expert` question to verify directly
in AppsFlyer's dashboard, not something resolvable from this data alone —
but it's a much smaller, well-posed question than "why is platform 8x
higher than attributed," which was really two unrelated problems (install
blending + duplicate tracking) disguised as one.

### Re-confirmed 2026-10-06, by network, 30-day window (2026-09-06 to 2026-10-05) — ranking holds, magnitude doesn't

Re-ran the full correction procedure on firstclub-v2, this time segmented by
`segments.ad_network_type` rather than account-wide, to answer a real
question: is Content's apparently-worst raw CPA an artifact of install
blending, or real? **It's real.** Raw and corrected rankings are identical
(best→worst: Search, Search Partners, YouTube, Content) — only the
*magnitude* of the gap changes, and it narrows (Content was 3.36x worse than
Search on the raw blended number; 2.59x worse once corrected). Don't read a
narrowed ratio as "not that bad" — Content remains clearly the worst channel
on real post-install purchases, just not as dramatically worst as the raw,
install-inflated number implied.

| Network | Spend (₹) | Raw "actions" | Raw CPA (₹) | Corrected actions | Corrected CPA (₹) | Corrected as % of raw |
|---|---|---|---|---|---|---|
| Search | 36,11,755 | 95,089 | 37.98 | 6,238 | 578.99 | 6.6% |
| Search Partners | 14,157 | 311 | 45.52 | 17 | 832.78 | 5.5% |
| YouTube | 49,05,900 | 47,863 | 102.50 | 4,708 | 1,042.03 | 9.8% |
| Content | 95,86,742 | 75,069 | 127.49 | 6,401 | 1,497.69 | 8.5% |

Only ~8% of the raw `conversions_platform` total across the account is a
real, single-tracked purchase — the rest is installs plus the duplicate
Firebase tracker described above.

**Two things changed since the September write-up, re-confirmed live rather
than assumed:**

1. **The canonical conversion-action id is not static — it already drifted
   once.** Google auto-created a new conversion action
   (`7378949446`, "FirstClub : Quality in Minutes (Android) in_app_purchase")
   with an updated Play Store listing title; the old one is gone from the
   live set. Querying by a hardcoded id will silently go stale. **Filter by
   stable characteristics instead**:
   `third_party_app_analytics_settings.provider_name = 'AppsFlyer'` AND
   `category = 'PURCHASE'` AND `primary_for_goal = true` AND
   `status = 'ENABLED'`, explicitly excluding any `type` starting with
   `FIREBASE_`. Also always verify by `appId`, not display name — the
   Firebase duplicate on this account has a *more specific-looking* display
   name ("FirstClub : Quality in Minutes") than the real AppsFlyer tracker,
   which would mislead a name-based check.
2. **Customer 9123477976 ("FirstClub Technology Private Limited") is not a
   pure Android account despite its naming** — it has grown a live iOS
   AppsFlyer purchase tracker (`7418320022`) alongside the Android one. Don't
   assume the two Google Ads customer ids map 1:1 to "iOS connection" /
   "Android connection" by name; check `conversion_action.app_id` per
   tracker.

**New GAQL gotcha** (add to the existing trap list below):
`segments.ad_network_type` + a conversion-action segment
(`segments.conversion_action_name`/`category`) together with
`metrics.cost_micros` in the same query fails with
**`PROHIBITED_SEGMENT_WITH_METRIC_IN_SELECT_OR_WHERE_CLAUSE`** — cost and
conversion-action segmentation aren't selectable together. Split into two
queries (conversions by network+action with no cost field; cost by network
alone with no conversion-action segment) and join client-side.

**Caveat for whoever acts on the corrected numbers**: don't disable or
deprioritize the Firebase-sourced purchase tracker (`7503566849`) inside
Google Ads itself just because it should be excluded from *dashboard
reporting* math — if this account's bidding strategy (tROAS/tCPA) depends on
Firebase-sourced conversion value for training, removing it from
`include_in_conversions_metric` could affect bid-strategy eligibility. That's
a question for whoever owns the bid strategy; this correction should live in
the reporting/query layer, not as a change inside Google Ads.

### Re-confirmed 2026-10-07 — both-primary-for-goal is necessary but not sufficient for double-counting; check the per-campaign override too

The account-level finding above (both `7378949446` AppsFlyer and `7503566849`
Firebase live, `primary_for_goal: true`, `include_in_conversions_metric: true`)
reads as if Google's bidding algorithm must be double-counting every purchase
for any campaign on this account. **Tested directly on the two Target CPA
campaigns analyzed this session, and it isn't happening for either of them:**

- `campaign_conversion_goal` returns **zero rows** for both campaigns — this
  account's App campaigns don't use the modern conversion-goals framework at
  all. An empty result here isn't an error; it's itself diagnostic that the
  campaign relies on the **legacy** mechanism instead.
- The legacy mechanism is `campaign.selective_optimization.conversion_actions`
  — both campaigns checked have an explicit override that already excludes
  the Firebase action: campaign `23591909679` counts only
  `[7378949446, 7278810661 (DOWNLOAD)]`; campaign `24040510982` counts only
  `[7378949446]`. Neither includes `7503566849`.
- Empirically confirmed, not just inferred from the override: a live,
  per-action query for both campaigns over the same 30-day window shows
  **zero** attributed conversions from the Firebase action on either campaign
  — so even account-level eligibility aside, Firebase contributed nothing to
  count twice here.
- Live `metrics.conversions` (Google's own raw, unfiltered, bidding-facing
  number) matches the category-filtered, single-canonical-action number
  almost exactly once `DOWNLOAD` is excluded (campaign `23591909679`:
  182,481 raw → 11,724 once installs are excluded, 100% from the AppsFlyer
  action; campaign `24040510982`: 4,596 raw, already all purchase-category,
  100% from the AppsFlyer action) — consistent with no duplicate-tracker
  inflation, only the already-known install-inclusion effect.

**Practically: check `campaign.selective_optimization.conversion_actions` per
campaign before concluding double-counting is live for it.** Both-primary
at the account level is the precondition, not the proof — a campaign with its
own override (as both of these have) can be clean even while the account-level
risk remains real for any *other* campaign that falls through to the
account-level `customer_conversion_goal` without its own override (a newly
auto-created campaign, for instance). Re-run this check per campaign, not
once for the whole account.

### Which named column is closest, definitionally — and why it differs by campaign subtype

Setting the messy real-numbers case aside, ranked by definition alone:

1. **In-app actions** — closest. Google's own documentation is explicit
   (confirmed directly on
   [support.google.com/google-ads/answer/13278587](https://support.google.com/google-ads/answer/13278587?hl=en),
   2026-09-20 — official wording: "In-app actions are a type of
   Conversions metric specific to App campaigns for engagement [ACE]…
   Installs [are] the Conversions metric for App campaigns for installs
   [ACI]"): for App Campaigns for Installs (ACI), **"installs are
   Conversions"** by default — "In-app actions" is the separate category
   reserved specifically for whatever happens *after* the install. That's
   the same conceptual scope as postclick conversions (always a post-install/
   post-click downstream action, never the install itself), regardless of
   which specific event it is — purchase, signup, subscription. It doesn't
   assume a category the way "Purchase conversions" does.
2. **Purchase conversions** — a narrower special case of #1, correct only
   when the client's actual postclick event is a purchase.
3. **Conversions (plain)** — for ACI, defaults toward *installs*, the
   wrong side of the click entirely, unless the campaign has been
   specifically configured with an in-app action as its primary goal (see
   below — this is a per-campaign fact, not guaranteed).
4. **All conv. / Participated in-app actions** — both definitionally
   broader-than-one-outcome by design (secondary actions + view-throughs;
   shared cross-campaign credit) — further away, not closer.
5. **Current model / by conv. time** — not comparable on this axis; timing
   and crediting lenses on the same underlying events, not different
   definitions of what's counted.

**"Conversions" vs. "In-app actions" converges for different reasons
depending on campaign type — do not assume both converge the same way:**

- **App Campaigns for Engagement (ACE)**: these target existing app users
  (deep links) — there is no install event in an ACE campaign at all, so
  "Conversions" is *structurally* always in-app-action-based; there's
  nothing else it could mean. **Confirmed empirically across several
  accounts including firstclub-v2 (2026-09-19): Conversions and In-app
  actions are consistently near-identical for ACE campaigns.** For ACE,
  either name is safe to use interchangeably.
- **App Campaigns for Installs (ACI)**: convergence is a *configuration
  fact*, not a guarantee. On firstclub-v2, ACI Conversions happens to
  equal In-app actions because these specific campaigns are configured
  with the in-app purchase as their primary goal (confirmed via
  `conversion_action.primary_for_goal`) — not because ACI structurally
  requires it the way ACE does. A different client running a pure
  install-volume ACI campaign would see Conversions track installs and
  diverge sharply from In-app actions. **Always check which goal a
  specific ACI campaign is actually configured for before assuming
  Conversions and In-app actions are interchangeable** — never assume it
  the way you safely can for ACE.

### The correct query, and exactly how far down it reaches

The right way to compute "In-app actions" is `metrics.conversions` filtered
to exclude the install category directly in the query — not a separate
metric field:

```sql
SELECT
  campaign.name,
  ad_group.name,
  segments.conversion_action_name,
  segments.conversion_action_category,
  metrics.conversions
FROM ad_group
WHERE segments.date BETWEEN 'YYYY-MM-DD' AND 'YYYY-MM-DD'
  AND segments.conversion_action_category != 'DOWNLOAD'
  AND metrics.conversions > 0
```

(Swap `ad_group` for `campaign` or `ad_group_ad` for a coarser/finer grain —
see the table below. `ad_group_ad` requires `conversion_action_category` in
the `SELECT` list too, or GAQL rejects the query with
`EXPECTED_REFERENCED_FIELD_IN_SELECT_CLAUSE`.)

**A dedicated-looking field is not the same as the right field — a real
mistake caught mid-session, worth the warning.** `metrics
.biddable_app_post_install_conversions` exists, matches the category-
filtered number exactly at campaign and asset grain, and looks like a
cleaner one-field answer — but it's very likely a bidding-algorithm-
internal metric (note the `biddable_` prefix), not what the Google Ads
UI's own "In-app actions" column is backed by. **It fails outright at
ad-group grain** (`PROHIBITED_METRIC_IN_SELECT_OR_WHERE_CLAUSE`,
naming `AD_GROUP` as unsupported) — confirmed live — **despite Google's own
`GoogleAdsFieldService` schema listing `ad_group` in its
`selectable_with`.** Real UI users see "In-app actions" broken out by ad
group with no trouble (confirmed against a live account), which only the
category-filtered `metrics.conversions` approach reproduces correctly.
**`selectable_with` describes field-level compatibility, not proof a query
actually works at that resource — every grain claim needs a live test, not
just a schema lookup.** This is the second time this exact trap has bitten
this investigation (the first: `segments.conversion_action_category`
listed compatible with asset-level metrics in general, but incompatible
specifically with `ad_group_ad_asset_view` — see below).

**Grain availability, confirmed live, 2026-09-12 to 2026-09-18
(firstclub-v2), category-filtered `metrics.conversions`:**

| Grain | Works? | Evidence |
|---|---|---|
| Campaign | ✅ | — |
| Ad group / adset | ✅ | 30 rows, sum 3,746 |
| **Ad** (`ad_group_ad`) | ✅ | 30 rows, sum 3,748 (real `ad.id`, still no `ad.name` — consistent with the ad-name finding at the top of this skill) |
| **Asset** (`ad_group_ad_asset_view`) | ❌ | `PROHIBITED_SEGMENT_IN_SELECT_OR_WHERE_CLAUSE` — `segments.conversion_action_category` is flatly incompatible with this resource, confirmed live, not a workaround-able syntax issue |

**The `ad_group_ad_asset_view` resource itself is a dead end for category-
filtered data — but there is a real workaround, via a segment instead of
the dedicated resource.** `segments.asset_interaction_target.asset`,
queried through the ordinary `campaign` resource (not
`ad_group_ad_asset_view`), returns real per-asset, category-filtered
In-app actions — confirmed live, 340 rows, no incompatibility:

```sql
SELECT
  campaign.name,
  segments.asset_interaction_target.asset,
  segments.conversion_action_category,
  metrics.conversions
FROM campaign
WHERE segments.date BETWEEN 'YYYY-MM-DD' AND 'YYYY-MM-DD'
  AND segments.conversion_action_category != 'DOWNLOAD'
  AND metrics.conversions > 0
ORDER BY metrics.conversions DESC
```

This returns an opaque asset resource name (e.g.
`customers/9123477976/assets/381603169668`), not the asset's actual
content — a **second query** against the `asset` resource resolves it:

```sql
SELECT asset.id, asset.type, asset.text_asset.text,
       asset.image_asset.full_size.url
FROM asset WHERE asset.id IN (381603169668, 381502948187)
```

(Field to select depends on `asset.type` — `text_asset.text` for `TEXT`,
`image_asset.full_size.url` for `IMAGE`, `youtube_video_asset
.youtube_video_id` for `YOUTUBE_VIDEO`.) **Confirmed end-to-end on real
data (2026-09-12 to 2026-09-18)**: the top-performing headline by actual
In-app actions (not installs) was **"Protein For You & Your Family"**
(180), ahead of "Kid-Safe, Mom-Approved" (166) — a genuine, working
creative-performance answer for a Google App Campaign that this
dashboard's own Creative Studio (Meta-only) cannot currently show at all.

One real limitation of this path: `segments.asset_interaction_target
.interaction_on_this_asset` (the asset's *type*/role — headline vs.
description vs. image) is queryable alongside `.asset` with no error, but
**never actually returns a value** — confirmed by inspecting the raw
`fieldMask` (it lists the field as requested) against the response (it
never appears in a single row). So this path gives you *which asset*
drove real purchases, but not *what kind* of asset it is directly — you
get that from the second query's `asset.type` field instead, same as the
content itself.

**So the corrected picture, superseding the "real ceiling is Ad" framing
above**: there is no clean single-query path to category-filtered,
asset-level data via the dedicated asset-view resource — but there is a
two-query path (segment + asset-resource lookup) that gets there anyway.
Don't conclude a capability is impossible just because the first,
most-obvious resource for it fails; check whether the same data is
reachable as a *segment* on a different resource before ruling it out.

### Standing check: in-app-actions vs. postclick, at the overall level, on every diagnosis — 25% is the escalation threshold

This is a mandatory step on **every** Google UAC diagnosis, not just a
reactive check when something already looks broken. Postclick conversions
(Branch/AppsFlyer/Adjust) are what the client's business actually cares
about, but they are not what Google's bidding algorithm optimizes toward —
Google bids toward its own in-app-actions signal (the correctly
category-filtered, correct-conversion-action number from the procedure
above, never the raw unsegmented `conversions_platform`). Since the two
numbers measure genuinely different things, a gap between them is normal —
but the size of that gap is itself diagnostic, and needs a defined
threshold, not a judgment call made fresh every time:

1. Compute the correctly-filtered in-app-actions number (category-filtered,
   matching whichever conversion action(s) `campaign.selective_optimization
   .conversion_actions` / `campaign_conversion_goal` / `customer_conversion_goal`
   actually mark biddable for that campaign — see the confirmed procedure
   above; never the raw `conversions_platform`, which mixes in installs and
   overstates any gap).
2. Compare it against postclick `conversions_attributed` for the same
   window, **at the overall campaign level, not ad-group level.** This is
   deliberately the opposite grain from failure pattern #8 above: an
   ad-group-level gap is frequently explainable by that ad group's own
   network mix (the confirmed supermoney-v2 YouTube finding) without
   indicating anything wrong — but a large gap that survives all the way up
   to the whole-campaign aggregate, averaged across every ad group and
   network split inside it, is a much harder signal to explain away as
   normal attribution-model noise.
3. **If the overall-level gap exceeds 25% (either direction) after correct
   filtering, hand it to `martech-expert` as an instrumentation-gap
   investigation** — a join-key mismatch, an attribution-window mismatch
   between the Google conversion action's configured window and the MMP's
   own lookback, a postback/tracking problem, or something else genuinely
   broken in the pipeline. This is a different question from failure
   pattern #8: #8 is about whether one ad group's Branch gap should change
   a pause/optimization decision; this is about whether the campaign's
   aggregate gap is large enough to indicate a data-integrity problem worth
   fixing regardless of what optimization decision is currently in play.
4. **Below 25%, treat the residual gap as ordinary cross-system attribution
   noise** (view-through credit the postclick source doesn't count, minor
   sync lag, model differences) rather than something to chase — the
   confirmed supermoney-v2 case above is the reference example: a real,
   fully-explained, YouTube-driven gap that needed no pipeline fix.

**Use `first_transactions` directly for the postclick side, not
`conversions_attributed`.** Confirmed live (2026-09-27):
`transformService.js:460-462` silently substitutes install count for FT
whenever Branch's raw data omits the `first_transactions` key for a
zero-FT day/ad-group — so `conversions_attributed` can be meaningfully
inflated with no error or flag (confirmed case: a campaign's FT read ~25%
high because of this). This is a real, codebase-wide bug affecting more
than this check, flagged separately to `developer` — until it's fixed,
this standing check specifically must sum `first_transactions` from the
raw postclick data, not read `conversions_attributed` off the `campaigns`
table.

**Confirmed live (2026-09-27, supermoney-v2, 13-26 Sep window)**: ACI
tripped the threshold, ACE didn't.

| Campaign | Google purchases (correctly filtered) | Branch FT | Gap | Verdict |
|---|---|---|---|---|
| ACI `Ch-UAC_Opt-BRFirstTran` | 183,791 | 145,173 | **+26.6%** | Over 25%, investigated |
| ACE `Ch-UACe_Opt-BRFirstTran_Acquisition` | 78,185 | 79,850 | -2.1% | Under 25%, noise |

**CORRECTION (2026-10-10) — the 26.6% figure above does not reproduce,
and should not be trusted as a baseline.** A later run, recomputing the
exact same Sep 1-26 window the same way (Google action 6878071183,
category-filtered, vs. Branch `first_transactions`), got **34.4%**, not
26.6% — a materially different number from the identical query. The
discrepancy is specifically in the click-date-based Google total; nobody
has been able to reconstruct what produced the original 183,791. **Don't
cite the 26.6%/183,791 figures as fact.** By contrast, this entry's own
conversion-date figure below (33.4%) turned out close to the later run's
independently-recomputed 33.8% for the same window — conversion-date was
right even though click-date wasn't. See the "Confirmed live
(2026-10-10)" entry further below for the full corrected picture,
including why this happened and the process fix.

**Open question this run surfaced: which side is the base.** Measured
against Google instead of Branch, ACI's gap reads 21.0% instead of 26.6% —
this skill doesn't yet specify which denominator to use, and it changes
whether a borderline case crosses 25% or not. Use `(Google − Branch) /
Branch` (Branch as base) until this is deliberately decided otherwise, for
consistency with the number reported here. Also note: measuring by
conversion date instead of click date pushed ACI to +33.4% — the 26.6%
figure understates the steady-state gap, since Google keeps attributing a
click's purchases up to 7 days later and the most recent 1-2 days of any
window will look artificially better than they'll settle. **(Corrected
2026-10-10: this reasoning about click-date lag was right, but the base
26.6% figure it was correcting from was itself wrong — see the correction
above and the full entry below.)**

**Root cause, confirmed, not a pipeline defect**: ACI competes with other
ad networks for the install itself; Branch's last-touch model gives the
install (and so the first transaction) to one winning network, while
Google's own report counts a purchase within its attribution window after
*either* a click or an engaged view on ACI's ad, regardless of which
network Branch credited the install to. Breaking the gap out by network
confirmed this precisely: Search +10%, Display +21%, **YouTube +120%** —
about 58% of ACI's total excess purchases come from YouTube, and ~92% of
the gap traces to Google's engaged-view credit specifically. ACE is the
control that confirms this isn't Branch generally rejecting view-based
credit: ACE has even more engaged-view-credited purchases than ACI and
still matches Branch within 2%, because ACE retargets existing app users
with no install to contest. **Not independently confirmable from this
codebase's data alone** — settling it for certain needs a Branch admin to
compare Branch's engaged-view attribution window against Google's, or run
a Branch multi-touch/overlap report for ACI's installs. If confirmed, there
is nothing to fix — it means ACI's own bidding signal (and its Google-side
reported CPA) is running ~27% more favorable than its real Branch-based
CPA, which the growth-plan/optimization numbers should account for.

**Confirmed live (2026-10-10, supermoney-v2) — the corrected picture, and
why the 26.6% figure was wrong.** Asked whether ACI's gap had been
trending upward (it had been reported as 26.6% on 9/27, then 28.3% in a
2026-10-10 campaign audit). It has not — **recomputing both historical
points the exact same way shows the gap flat to falling, not rising**:

| Week (Sat-Fri, conversion date) | Gap |
|---|---|
| Sep 5-11 | 34.4% |
| Sep 12-18 | 31.9% |
| Sep 19-25 (last full-geo week) | 35.7% |
| Sep 26-Oct 2 (geo-carve-out transition) | 31.8% |
| Oct 3-9 (fully carved out) | 26.9% |

Sep 1-26 recomputed fresh: **34.4%** by click date (not the original
26.6%), **33.8%** by conversion date (close to the original entry's own
33.4% secondary figure, which held up fine — it was specifically the
click-date headline number that was wrong). Nobody could reconstruct what
produced the original 183,791/26.6%; treat it as an unexplained
measurement error, not a real reading from a different moment in time.

**Two traps caused this, and both are general, not one-off:**
1. **Click-date vs. conversion-date, for any window ending near "now."**
   Google keeps crediting a click's purchases for up to ~7 days after the
   click, so a window ending within a week of the day you're running the
   query will show an artificially low Google count on its most recent
   days — making the gap look *smaller* than it really is, right when
   it's most tempting to treat a dip as a trend. **Always use
   `metrics.conversions_by_conversion_date` instead of
   `metrics.conversions` for any window ending within 7 days of today.**
2. **`conversions` vs. `all_conversions` — never mix them between two
   checks you intend to compare.** These differ by 7-13% on this action,
   and the ratio has drifted over time (~1.07 in April, ~1.13 from July
   onward) — silently swapping which one a calculation uses between two
   points in time manufactures exactly the kind of false "trend" this
   correction is unwinding.

**Longer-range pattern (real, not yet proven causal): the gap tracks
ACI's spend level**, not time — about 18-24% at low spend (₹0.9-1.6M/week,
Jul-Aug), about 30-36% as spend scaled back up (₹3.4-4.7M/week,
late Aug onward). One plausible reading: at higher budgets Google reaches
further into inventory (more YouTube) where it credits purchases Branch
doesn't — consistent with the YouTube root cause above — but this is a
correlation, not confirmed as the mechanism.

**The 9/25-29 geo carve-out (five states moved to the Firebase campaign)
does not explain an upward trend — if anything the post-carve-out gap
reads lower.** Branch can't split ACI's results by state to check this
directly, but bounding it: those states were a steady 9-11% of ACI's
Google-side volume every week before the carve-out; removing them would
need their own gap to differ from the rest of India by ~50 points to
shift the blended gap by the ~5 points observed, which is implausible. A
rough combined ACI+Firebase-campaign line (not a true like-for-like,
since the Firebase campaign optimizes toward a different conversion
action) shows the same falling pattern, not a rising one.

**Standing process fix, to stop this kind of error recurring**: whoever
runs this check should (a) use `conversions_by_conversion_date` for any
window ending within 7 days of today, (b) never mix `conversions` and
`all_conversions` between two numbers being compared, and (c) **always
quote the raw Google and Branch totals alongside the gap %**, not just
the percentage — the original 26.6% entry only recorded the final
percentage and one total, which is exactly why this error went
undetected for two weeks and couldn't be traced back once found.

### Other segments tested against the category-filtered query

Confirmed live, 2026-09-12 to 2026-09-18, firstclub-v2 — useful as a
reference for what else composes cleanly with the In-app-actions query,
and what doesn't even though it's technically queryable:

| Segment | Works? | Notes |
|---|---|---|
| `segments.ad_network_type` (network) | ✅ Useful | Via `campaign` directly. Real Search/Content(Display)/YouTube/Search Partners breakdown of actual purchases — not just impressions/clicks, which is what the existing "network" gap note (above, under "What Google can report") only covered before |
| `segments.geo_target_state` | ✅ Useful | **Not** via `campaign` (flatly incompatible, confirmed unconditionally — unrelated to the category filter). Works via the dedicated `geographic_view` resource instead. Returns a `geoTargetConstants/NNNN` id needing a second lookup against `geo_target_constant` (`.name`, `.canonical_name`) to become human-readable — confirmed resolving correctly (a Hyderabad campaign's spend correctly resolved to "Telangana," a real sanity check that the whole chain works) |
| `segments.geo_target_city` | ✅ Useful | Same pattern as state — via `geographic_view`, same two-query resolution needed |
| `segments.hour` | ✅ Useful | Via `campaign` directly, real hour-by-hour (0-23) breakdown, no resolution step needed |
| `segments.ad_destination_type` | ⚠️ Queryable, not useful for ACI | Every value returned is `UNKNOWN` or `NOT_APPLICABLE` — this segment is meaningful for Search/Display ads with a real landing-page destination; App Campaigns don't have one. **Worth re-testing when building a Search/Display/Pmax campaign-type skill later** — likely far more useful there |
| `segments.click_type` | ⚠️ Queryable, not useful for ACI | Only 5 rows, every one `URL_CLICKS` — App Campaigns don't have the click-type variety Search ads do (no call clicks, sitelink clicks), so nothing to differentiate |
| `segments.asset_interaction_target.asset` | ✅ Useful | See above — the asset-level workaround |
| `segments.asset_interaction_target.interaction_on_this_asset` | ❌ Queryable but empty | See above — never returns a value in practice |

### CTR/CPC diagnose a network problem; conversion volume is what sizes the fix — don't stop at the rate metrics

**Confirmed live (2026-10-10, supermoney-v2).** `segments.ad_network_type` composes
with the category-filtered in-app-actions query above — meaning real
conversion *volume* by network (Search/Display/YouTube), not just CTR/
impressions/clicks, has been reachable in this skill the whole time. Using
it changes lever-sizing answers, not just diagnosis detail:

- CTR and CPC tell you a network's engagement/cost-efficiency is
  decaying — that's real and worth acting on, but it says nothing about
  how much actual conversion volume sits behind the trend. The same CTR
  drop can mean very different things depending on the volume underneath
  it.
- **A real lever estimate built on CTR/CPC alone was off by roughly 2x**:
  a creative-refresh lever initially sized at +800-1,000 FT/day from
  YouTube CTR (-28%) and Display CPC (+22-32%) data was cut to +250-600
  once actual network-level conversion volume showed the campaign hadn't
  lost volume on those networks at all — it was paying more for
  **roughly the same volume**, a materially different (and smaller)
  problem than "lost conversions" implied.
- **A second lever's recommendation flipped entirely once volume was
  checked**: a campaign that looked attractive to revive purely on its
  historical CPFT turned out, once its historical conversion volume was
  broken out by network and cross-referenced against the live campaign's
  own volume, to be drawing 90% of its YouTube conversions from creative
  already running live elsewhere, and 63% of its Search conversions were
  directly substitutional with the live campaign's own Search volume —
  something no CTR/CPC trend could ever surface, because overlap and
  substitution are volume questions, not rate questions.

**The standing rule: CTR/CPC is for diagnosis, network-level conversion
volume is for sizing.** Treat any lever estimate built only on CTR/CPC as
provisional — pull actual conversion volume by network (the query above)
before presenting a sizing number, not just when something already looks
broken enough to investigate further.

### Ad Strength — a real, live, per-ad quality rating (resolves a gap flagged earlier this session)

**Confirmed live (2026-09-19)**: Google Ads has a real "Ad Strength for App
Campaigns" rating — per
[Google's documentation](https://support.google.com/google-ads/answer/9777426),
this rates an ad's effectiveness based on the **number and variety of
assets** it includes (headlines, descriptions, images, videos), on a
five-level scale: **Incomplete → Poor → Average → Good → Excellent**
(`Incomplete` specifically means under 2 headlines / 1 description —
the account hasn't met minimum requirements yet). Not the "Low/Good/Best"
label loosely assumed earlier in this skill before verification — that
assumption was wrong; the real scale is wider and named differently.

**The field**: `ad_group_ad.ad_strength`, on the same `ad_group_ad`
resource already established as the real (if unnamed) ad-level grain for
App Campaigns. Confirmed populated with real values across dozens of
firstclub-v2's App Campaign ads (`EXCELLENT`, `GOOD`, `AVERAGE`, `POOR`
all observed live) — this is not a Search/Display-only concept, contrary
to what might be assumed given App Campaigns' automated, no-manual-ad
structure.

**Composes cleanly with the category-filtered In-app-actions query** —
confirmed live, real Ad Strength rating alongside real In-app actions
count in one query:

```sql
SELECT campaign.name, ad_group.name, ad_group_ad.ad_strength,
       segments.conversion_action_category, metrics.conversions
FROM ad_group_ad
WHERE segments.date BETWEEN 'YYYY-MM-DD' AND 'YYYY-MM-DD'
  AND segments.conversion_action_category != 'DOWNLOAD'
  AND metrics.conversions > 0
```

On firstclub-v2, the top In-app-actions ad groups this window all carried
`EXCELLENT`/`GOOD` ratings (Mixed_Shoppers: 746 conversions, EXCELLENT;
Kids: 628, EXCELLENT; Protein: 595, GOOD) — a real, if unsurprising,
internal-consistency signal, not proof of causation.

**One related field is a dead end for this campaign type, confirmed by
checking rather than assuming**: `asset_group.ad_strength` also exists in
the schema, but querying the `asset_group` resource against firstclub-v2
(`campaign.advertising_channel_type = 'MULTI_CHANNEL'`) returns **zero
rows** — `asset_group` is the Performance Max/Demand Gen resource, not
something App Campaigns populate at all. Use `ad_group_ad.ad_strength`,
never `asset_group.ad_strength`, for this campaign type.

**The benchmark, and a real account checked against it**: Google's
[best-practices guidance](https://support.google.com/google-ads/answer/14104492)
says Ad Strength should be **"Good" or better**. Checked live against
firstclub-v2's full set of enabled App Campaign ads (2026-09-19): only
**34.4% (32 of 93) meet that bar** (14 `EXCELLENT` + 18 `GOOD`, vs. 22
`POOR`, 7 `AVERAGE`, 32 unrated). Query used:

```sql
SELECT campaign.name, ad_group.name, ad_group_ad.ad.id,
       ad_group_ad.ad_strength, ad_group_ad.status
FROM ad_group_ad
WHERE campaign.advertising_channel_type = 'MULTI_CHANNEL'
  AND ad_group_ad.status = 'ENABLED'
```

**Not uniform across campaigns — and it lines up with a pattern already
documented elsewhere in this skill.** `FC_Perf_Google_NU_App_Android_
25022026` (the account's top-volume campaign, and the one with the
cleanest single-tracker purchase data) sits at 87.5% Good-or-better
(21/24). `FC_Perf_Google_RnB_App_Android_17072026` (used throughout this
skill as the "clean attribution" example campaign) sits at **7.7%
(1/13)** — 12 of its 13 ads are `POOR`. Creative quality is a real,
independent variable worth checking before assuming a weaker campaign's
underperformance is purely an attribution/tracking story — RnB's clean
tracking doesn't mean its creative is fine, and this account shows both
can be true (or false) independently of each other.

## Other confirmed App Campaign best practices (Google's own guidance, 2026-09-19)

From the same
[best-practices reference](https://support.google.com/google-ads/answer/14104492),
not yet independently verified against live account behavior beyond the
Ad Strength check above:

- **Target only one in-app action per tCPA campaign** — don't spread a
  single campaign's optimization across multiple events; pick the one
  that matters and mark it primary (see the `conversion_action.primary_
  for_goal` discussion above).
- **Auto-tagging is described as "a critical requirement for app
  conversion measurement"** — worth a `martech-expert` check on any
  account showing unexplained conversion gaps; not yet confirmed whether
  it's actually enabled/verified on firstclub-v2.
- **Conversion windows should be ≤ 30 days for in-app actions.**
- **Attribution sharing is automatically enabled by default across ACI,
  ACE, ACpre, and deep-linked campaigns** — confirmed against Google's
  dedicated documentation, see above. The actionable check when these
  types run simultaneously in the same account isn't "enable it," it's
  confirming a client hasn't manually turned it off under Settings →
  Conversions → App attribution sharing — directly relevant to the
  "Participated in-app actions"/"Participated Installs" section above.
- **A third App Campaign subtype exists**: ACpre (App Pre-Registration) —
  not covered anywhere else in this skill; out of scope until a client
  actually runs one.

## iOS vs Android — not a dashboard dimension, check the campaign name instead

**Confirmed against real data**: this dashboard's `device` dimension is
not populated for Google Ads data at all (0 of 246 rows for
firstclub-v2, Sept 11). This isn't specific to one client — `device`,
along with `campaignType`/`campaignSubType`/`gender`/`initiative`, is a
Shein-era legacy dimension no current client's data populates, and this
dashboard now hides that pivot pill entirely when a client has no real
values for it. **Do not expect a Device pivot pill to split a Google App
Campaign by platform — for any current client, it won't be there.**
(Unlike `gender` — see "What Google can report that this dashboard
doesn't pull yet" below — `device` isn't blocked at Google's API; it's
simply off in this client's pull config, and genuinely worth turning
on — see that section for why "App Campaigns are mobile-only" turned
out to be wrong when checked against real data.)

When a client runs separate iOS and Android App Campaigns (common
practice, not universal), the split shows up **in the campaign name
itself**, by whatever convention that client happens to use — confirmed
on firstclub-v2: `..._App_Android_...` / `..._App_iOS_...` substrings.
This is a per-client naming habit, not a platform guarantee — check the
actual campaign names for the client in front of you rather than
assuming a fixed pattern (KlinqV1 and SoulSensai V2, for comparison, run
Search/Demand Gen campaigns with no iOS/Android split in the name at
all, because they're not running App Campaigns in the first place).

If a client's iOS App Campaigns show unusually sparse or delayed
conversion data compared to Android, that is expected and platform-
level, not a tracking bug — Apple's SKAdNetwork constrains iOS install
attribution to a limited, sometimes-delayed, privacy-thresholded
postback rather than App Campaigns' normal real-time reporting; that
divergence is a real platform limitation, not a `martech-expert` bug to
chase in this dashboard's pipeline.

## Which pivot dimensions are actually usable for an App Campaign

**Two separate config knobs control this, confirmed by reading
`googleAdsFetcher.js` alongside `pivotDimensions.js`, not just one:**

1. `connectorConfig.google_ads.dimensions` (per-connector "pull config")
   — decides what actually gets **fetched from Google** and written to
   `preclick_facts`. This is `buildCuratedQueries()`'s input.
2. `connectorConfig.reportingDimensions` (client-wide) — decides which
   pivot **pill renders in the UI** at all (`dimOptionsFor()`).

They usually get set to the same list by whoever configures a client
(firstclub-v2's are both `["campaign","adset","ad"]`), but they don't
have to match, and a mismatch produces two different symptoms: a
dimension in (1) but not (2) is fetched and stored but never shown; a
dimension in (2) but not (1) shows an empty pill because nothing was
ever pulled for it. Check both before concluding a dimension is
"impossible" — it may just be off in one of the two places, which is a
config fix, not a platform limitation. See the next section for which
dimensions are worth turning on that way, versus which are genuinely
blocked at Google's own API for this campaign type.

Net for a pure Google App Campaign client on today's default config
(firstclub-v2): your real usable breakdown is **Campaign × Ad set ×
Date**, cut by iOS/Android through the campaign name itself (see above).
**Ad** renders as a pill (it's in both dimension lists) but is
structurally empty for the reason in "Read this first." **Channel** is
always available but only useful for a cross-channel blend.

## What Google can report that this dashboard doesn't pull yet

Per Google's own App Campaign reporting documentation (cross-checked
against this dashboard's actual fetcher code, not assumed) — some gaps
above are genuine Google platform limits, others are just work this
codebase hasn't done. Don't conflate the two when talking to
`martech-expert`/`developer`: a platform limit needs a workaround or
acceptance, a pipeline gap needs a config change or new fetcher code.

**Genuinely blocked at Google's API for App Campaigns (not fixable by
config or new code in this dashboard):**
- **Keyword-level reporting** — completely unavailable for App Campaigns
  (unlike Search). `keyword_view` will return nothing for this campaign
  type regardless of config.
- **Audience/demographic segments** (age, gender, audience-list
  performance) — Google manages App Campaign targeting entirely through
  automated signals and does not expose a breakdown by who saw/converted.
  This is a stronger, platform-level reason for `gender`'s absence than
  "legacy Shein dimension nobody populates" (the other reason already
  documented) — for App Campaigns specifically, it isn't just unused,
  it's structurally unavailable.
- **Asset-combination performance** — you can see how one asset (a
  single headline/image/video) performed, but never the specific
  combination Google served together (e.g. "Headline A paired with
  Video B"). No resource exposes this at any campaign type.
- **Placement report** (`detail_placement_view` / `group_placement_view`
  — third-party apps/sites/YouTube channels an ad appeared on) — **fully
  resolved (2026-09-20), root cause confirmed via Google's own official
  announcement, not just live-testing.** Google deliberately deleted all
  App Campaign placement data from these two API reports (plus the
  legacy AdWords API's equivalent reports) **on 2022-01-05**, stating the
  removed data "didn't fully represent the complete view of the
  placements" needed for brand-safety review, and pointed API users to
  the **"App Campaigns Brand Safety Placement report," a Google Ads UI-
  only report with no API equivalent** ([googblogs.com summary of
  Google's announcement](https://www.googblogs.com/important-changes-to-placement-reporting-for-app-campaigns/);
  also covered by [Search Engine Land](https://searchengineland.com/google-removed-app-campaign-placement-data-from-ads-and-adwords-api-reports-381423)).
  This is the same pattern as "Participated in-app actions" above — a
  real report that exists in the Google Ads UI with **no queryable API
  path at all**, not a sync gap or a wrong-resource mistake. **If a
  client or an external source says this data is visible "in the Google
  Ads interface" for ACI/ACE, they are very likely right — and it does
  not mean the API can reach it.** Always ask which surface (UI vs. API)
  a placement claim is coming from before treating it as something this
  dashboard (or any Ads-API-based tool) could ever surface.
  **Empirically reconfirmed live (2026-09-20)** before finding the
  official source, as a real isolation test: both resources return real
  data for a Demand Gen campaign (`KlinqV1`, 100/100 sampled rows, real
  `placement_type: YOUTUBE_CHANNEL`) and zero rows for every App Campaign
  tested across two independent accounts (`supermoney-v2`'s ACI + ACE,
  `firstclub-v2`'s App Campaigns) — ruling out "this account has no
  placement data" as an alternative explanation. Also confirmed Pmax has
  its own dedicated resource, `performance_max_placement_view` (not
  `group_placement_view`/`detail_placement_view`), which populates real
  rows including `placement_type: MOBILE_APPLICATION` — and a full schema
  sweep of every `%placement%` field found no App-Campaign-specific
  equivalent anywhere. **A claim surfaced externally (2026-09-20)
  asserted `group_placement_view`/`detail_placement_view` are supported
  for ACi/ACe with `placement_type: MOBILE_APP`/`YOUTUBE_CHANNEL` values —
  contradicted by both live testing and Google's own 2022 announcement;
  treat it as wrong for the API, though it may be describing the UI-only
  Brand Safety report correctly.** This also fits App Campaigns' own
  model: fully automated, no manual placement targeting at all (see "What
  you can and can't control" below) — Google's stated reason for pulling
  this from the API (incomplete brand-safety representation) is more
  credible than "no placement control surface" alone would explain, since
  the UI report still exists specifically for brand-safety review, just
  not through the API.

**Available from Google, but this dashboard's fetcher doesn't query it
today (real, scoped gaps — flag to `developer`/admin, not `martech-expert`,
since nothing is misconfigured, it's just unbuilt):**
- **Network type** (`segments.ad_network_type` — Search, Google Play,
  YouTube, Discover, Display) — `googleAdsFetcher.js` already has the
  code path for this (`DIMENSION_FIELDS.network`, mapped through
  `toCanonicalNetwork`), it's just not in firstclub-v2's
  `google_ads.dimensions` pull config. **This is the cheapest of these
  gaps to close** — adding `'network'` to both `connectorConfig
  .google_ads.dimensions` and `connectorConfig.reportingDimensions`
  would surface a real Search-vs-YouTube-vs-Discover-vs-Display split
  for this client's App Campaigns with no new fetcher code, only a
  config change (though note Google explicitly does not expose
  sub-placements within a network, e.g. YouTube in-stream vs YouTube
  Shorts — network type is as fine as it gets).
- **Asset-level performance** (`ad_group_ad_asset_view` — per-asset
  clicks/impressions/conversions/cost, plus Google's own asset
  performance rating, per headline/description/image/video) — a
  genuinely different, real resource from the `ad_group_ad` query this
  fetcher already runs (which is why "no ad-level data" above is true for
  `ad_group_ad` specifically, not a claim that Google has nothing at any
  sub-campaign grain). Not queried anywhere in this codebase today. This
  is a bigger lift than the network fix — new fetcher query, new stored
  shape, new UI to display a rating — but it's the closest thing to
  Google-side "creative performance" data this campaign type actually
  offers, and would be the right answer if a client specifically wants
  asset-level insight beyond what Creative Studio (Meta-only, see below)
  provides. **Confirmed live (2026-09-13)** against firstclub-v2's real
  account: `ad_group_ad_asset_view` also composes cleanly with
  `segments.ad_network_type` in the same query — so a future fetcher
  could get asset-level performance broken down by network in one query,
  not two separate builds.
  **The rating field itself, `ad_group_ad_asset_view.performance_label`,
  confirmed live (2026-09-19) against firstclub-v2's real App Campaign
  assets — populated with real, varied values (`BEST`, `GOOD`,
  `LEARNING`, `LOW`, `UNKNOWN` all observed across 20 sampled rows), not
  a placeholder or always-`UNKNOWN` field.** Full enum (via
  `GoogleAdsFieldService`): `BEST`, `GOOD`, `LEARNING`, `LOW`,
  `NOT_APPLICABLE`, `PENDING`, `UNKNOWN`, `UNSPECIFIED`. This is the
  actual Google-side per-asset quality signal (distinct from
  `ad_group_ad.ad_strength`, which rates the whole ad's asset
  variety/completeness, not any one asset's individual performance) —
  the field to use if a future fetcher builds real per-asset creative
  performance for App Campaigns.
  **Creative orientation/format (vertical, square, horizontal) should be
  part of any creative-quality analysis on ACI/ACE, alongside Ad
  Strength and this performance_label — check whatever's actually
  available before assuming it isn't.** Not yet live-tested against a
  real account, so treat the following as unconfirmed until checked: for
  `IMAGE` assets, `asset.image_asset.full_size.width_pixels`/
  `height_pixels` (queried off the `asset` resource, same pattern as the
  asset-resolution lookup earlier in this skill) would let orientation be
  derived directly (width > height = horizontal, etc.) — this field has
  not been confirmed to populate for a real account and needs a live
  check before relying on it. For `YOUTUBE_VIDEO` assets, Google Ads'
  own API is not known to expose video dimensions/orientation at all
  (YouTube videos are referenced by `youtube_video_id` only) — likely
  needs a separate YouTube Data API lookup if this matters for video
  specifically, not something to assume is reachable from
  `GoogleAdsFieldService` alone. **Before reasoning about creative
  orientation for a real client, run the live check this paragraph is
  missing (query `asset.image_asset.full_size` on a real account's App
  Campaign image assets) rather than assuming either the field populates
  or the dashboard already has this data** — as of this writing, no
  asset-level data of any kind is ingested by this dashboard for Google
  Ads (see "No creative-level data exists for Google Ads" below), so an
  ad-hoc analysis needing this has to pull it live via `martech-expert`,
  not read it from stored data.
- **Conversion-action segmentation** (`segments.conversion_action` — split
  Install vs. specific downstream in-app events at Google's own layer)
  — not queried today. Lower priority than the two above: this dashboard
  already gets a similar distinction via postclick facts' own
  conversion-source event name (AppsFlyer/Branch/Adjust), a different but
  functionally overlapping path — worth knowing this exists on Google's
  side too, mainly for cross-checking postclick data against Google's own
  view when a `martech-expert` question comes up.
- **Geographic segmentation** (country/territory) — no code path in
  `googleAdsFetcher.js` fetches this at all currently, for any client.
- **Device** (`segments.device`) — fetchable today
  (`DIMENSION_FIELDS.device` already exists in the fetcher), and **not
  low-value the way this section used to claim**. **Corrected after a
  live GAQL query against firstclub-v2's real account (2026-09-13)**:
  `SELECT campaign.id, segments.device, metrics.impressions FROM
  campaign WHERE segments.date DURING LAST_30_DAYS` returned MOBILE,
  TABLET, *and* CONNECTED_TV for real App Campaigns, with meaningful
  impression volume on all three (one campaign: 87.2M MOBILE, 860K
  TABLET, 1.92M CONNECTED_TV). App Campaigns serve across YouTube/
  Display surfaces that render on tablets and connected TVs, not just
  phones — "App Campaigns are mobile-only" was a reasonable-sounding
  assumption that turned out to be wrong the moment it was checked.
  Turning `device` on (same two-config-knob fix as network, above) is
  worth doing, not a low-value nice-to-have.

## No creative-level data exists for Google Ads in this dashboard's Creative Studio

**Confirmed against real data**: `creative_assets` for firstclub-v2 has
352 rows for `meta_ads` and **zero for `google_ads`**. This dashboard's
Creative Studio / creative-fatigue analysis is Meta-only — there is no
ingested creative-level data for Google campaigns of any kind today.
**Never attempt to diagnose "which creative is fatigued" or "which asset
is underperforming" for a Google App Campaign using Creative Studio —
that data doesn't exist there.** This is a pipeline gap, not a Google
platform wall — see "Asset-level performance" above for the real
resource (`ad_group_ad_asset_view`) that could feed something like it if
ever built. Until then, any creative-level judgment for Google has to
come from Google's own UI/API directly, outside this tool.

## Other GAQL resources checked against real ACI/ACE campaigns (2026-09-20)

A broader schema sweep (all 183 `RESOURCE`-category fields in
`GoogleAdsFieldService`, cross-checked against what's already documented
above) turned up several resources not covered elsewhere in this skill.
Live-tested against supermoney-v2's real ACI (`Ch-UAC_Opt-BRFirstTran`,
id `21658820526`) and ACE (`Ch-UACe_Opt-BRFirstTran_Acquisition`, id
`22403871002`) campaigns — the same account and campaigns used in the
budget-cut RCA in the failure-patterns list above.

**Real and populated — worth using:**

- **`recommendation`** — Google's own optimization suggestions, and for
  `CAMPAIGN_BUDGET` type, a real forecasted budget-impact curve
  (`campaign_budget_recommendation.budget_options`, each with a
  `potentialMetrics.conversions`/`cost_micros` projection). **Confirmed
  live**: the ACI campaign carried exactly one live recommendation —
  `CAMPAIGN_BUDGET`, current budget ₹670,000/day, with Google's own model
  projecting conversions at higher budget levels (₹670K→459,064 conv,
  ₹1.2M→614,364 conv, ₹2M→793,141 conv, ₹2.8M→938,457 conv — each
  option's own `cost` shows the model isn't assuming proportional-only
  scaling). **This is the right tool to *validate* a "campaign looks
  under-budgeted" hypothesis (failure patterns 3/5 above) with Google's
  own forecast instead of inference from historical cost/conversion
  curves alone** — query it before or alongside the budget-vs-target
  math check. ACE had zero recommendations at the same time (Google's
  model saw nothing to suggest for it) — absence of a recommendation is
  itself informative, not just presence.
  ```sql
  SELECT recommendation.type, recommendation.campaign, recommendation.dismissed,
    recommendation.campaign_budget_recommendation
  FROM recommendation
  WHERE recommendation.campaign = 'customers/{id}/campaigns/{campaignId}'
  ```
- **`customer_conversion_goal`** — account-level list of which conversion
  categories are configured as `biddable` (confirmed live: `DEFAULT`,
  `PURCHASE`, `DOWNLOAD`, all `origin: APP`, all `biddable: true` on
  supermoney-v2). A cleaner, single-query way to answer "which event
  categories can bidding even optimize toward at all" than enumerating
  `conversion_action` rows one by one — pair with `campaign_conversion_goal`
  (same shape, campaign-scoped) to check for a per-campaign override; on
  supermoney-v2's two campaigns that returned zero rows, meaning neither
  overrides the customer-level goals.
- **`third_party_app_analytics_link`** — account-level list of linked MMP
  connections. **Confirmed live**: 2 rows on supermoney-v2, each just a
  `shareable_link_id` (opaque, doesn't name the provider). Useful as a
  fast, direct **count** of how many third-party analytics links exist at
  the account level — e.g. to sanity-check a suspected duplicate-tracker
  situation (see the `conversions_platform` vs `conversions_attributed`
  section above) before doing the heavier `conversion_action` dig needed
  to identify *which* providers they are.
- **`change_status`** — a lighter companion to `change_event` (see
  `google-ads-api`'s Change History section): just
  `resource_type`/`resource_status`/`last_change_date_time`, no
  old/new values or `user_email`. **Confirmed live and populated** for
  both ACI/ACE campaigns. Cheaper for a first-pass "did anything change
  recently, and what kind of thing" scan across more resource types in
  one query than `change_event` covers by default — but reach for
  `change_event` (not this) as soon as you need the actual before/after
  value or who made the change.
- **`campaign_search_term_insight`** — a genuine, surprising exception to
  this skill's own "keyword-level reporting is completely unavailable for
  App Campaigns" claim above, and **richer than an initial pass over its
  bare fields suggested** — worth recording exactly what was corrected
  and how, since it's a real example of this skill's own "always live-test,
  don't infer from a resource's field list alone" discipline.
  **Confirmed live**: the top-level query (`category_label` only) returns
  320 rows for the ACI campaign, each a real theme like `"super money
  flipkart app download"` or `"send money app"` — genuine, specific query
  intent, not generic buckets. **An initial check of just the four bare
  fields on this resource (`campaign_id`, `category_label`, `id`,
  `resource_name`) wrongly concluded it carries no metrics to join
  against — checking `selectable_with` on the resource itself (not just
  its listed fields) showed real `metrics.clicks`/`conversions`/
  `impressions`/`search_volume` and `segments.search_term` all compose
  with it.** Three real gotchas once you actually query those, all
  confirmed live:
  - `segments.search_term` (the literal underlying query, not just the
    theme label) **cannot be combined with any metric field in the same
    query** — `PROHIBITED_SEGMENT_WITH_METRIC_IN_SELECT_OR_WHERE_CLAUSE`.
    Query them separately.
  - `segments.search_term` also **requires filtering to one specific
    insight's `resource_name`**, not just the campaign — a bare
    `campaign_id` filter 400s with `REQUIRES_FILTER_BY_SINGLE_RESOURCE`,
    and adding `LIMIT` alongside the segment 400s with
    `LIMIT_NOT_ALLOWED_WITH_SEGMENTS`. **Confirmed live**: filtering to
    one theme's exact `resource_name` correctly drilled down to 305 real
    literal search terms underneath it (`"flipkart app download"`,
    `"super money flipkart download"`, `"assured flipkart app"`, etc.) —
    so the two-step pattern (list themes → drill into one by
    `resource_name` for its literal terms) genuinely works, it's just
    gated behind these two restrictions.
  - `metrics.search_volume` is a **bucketed min/max range** (e.g.
    `1,000,000–10,000,000`), and reads as a market-wide query-volume
    estimate for the theme (Keyword-Planner-style), **not this
    campaign's own delivery on it** — confirmed by the fact that
    every one of this ACI campaign's 320 theme rows showed real
    `search_volume` ranges while `metrics.impressions/clicks/conversions`
    were all **exactly zero, with zero rows surviving a `metrics
    .impressions > 0` filter** (not a sort-order artifact — checked
    directly). The same metrics fields **do** carry real nonzero values
    at `customer_search_term_insight` grain on this same account (e.g.
    one theme: 1,467 clicks, 21,199 impressions) — **not independently
    confirmed whether campaign-level metrics are genuinely never
    populated for this campaign type, or just zero for this specific
    account/window**; treat campaign-grain metrics on this resource as
    unreliable until checked on another account, and use
    `customer_search_term_insight` if you need real attributed
    performance numbers rather than just the theme list.
  Net: this resource is a real, valuable **qualitative audience/intent
  signal** (what users were actually searching around this campaign's
  delivery, down to literal query text) — genuinely useful for creative
  and messaging alignment even though App Campaigns have no manual
  keyword targeting to act on it with directly. It does not (at least not
  reliably at campaign grain, confirmed on this account) give you
  performance-per-theme — don't present it as filling the
  keyword-performance gap the way a real Search campaign's keyword
  report would.

**Checked live, confirmed genuinely unavailable for this campaign type
(not a query mistake):**

- **`campaign_simulation`** — real resource, populates real data in this
  same account (one `BUDGET`-type simulation row, but only for the
  account's `PERFORMANCE_MAX` campaign, `SMB_Pmax-LeadGen_2505` — the
  same isolation pattern used for the placement-report finding above).
  **Zero rows for both the ACI and ACE campaigns**, filtered or
  unfiltered. Google's own bid/budget simulator ("what conversions would
  I get at bid X") simply isn't generated for these App Campaigns — use
  `recommendation` above instead for a budget-impact forecast on this
  campaign type.
- **`bidding_strategy_simulation`** — zero rows account-wide. Expected,
  not a gap: this resource simulates **portfolio** (shared) bidding
  strategies specifically, and both ACI/ACE here run a standard,
  campaign-level Target CPA, not a shared strategy — **not independently
  confirmed whether this resource would populate for an App Campaign
  that does run a shared bidding strategy**, since no such account was
  available to test.

**Checked live, no data on this account — inconclusive, not confirmed
either way:**

- **`experiment` / `experiment_arm`** — real resources, correct schema,
  but supermoney-v2 has zero active experiments of any kind right now,
  so this doesn't confirm or rule out App Campaign experiment support.
  Relevant to a real forum thread about a missing "Directional split"
  on an Android ACI video-only campaign — worth testing directly against
  an account that has an active App Campaign experiment before drawing
  any conclusion.

## What you can and can't control

App Campaigns are **fully automated** — there are no ad groups, no
keywords, no manual placement targeting. The levers that actually exist:

- **Budget** — the single biggest lever. Per
  [Google's own best-practices documentation](https://support.google.com/google-ads/answer/14104492)
  (not the vague "roughly 10-15x" this section used to say — that was an
  approximation; Google publishes precise, per-bidding-mode ratios):
  **daily budget should be at least 50x the bid for ACi tCPI** (target
  cost-per-install), **at least 10x the bid for ACi tCPA**, and **at
  least 15x the bid for ACe tCPA**. A campaign stuck at low volume with
  volatile CPI is very often simply under-budgeted for its own target,
  not a targeting problem — check the actual ratio against the right
  number for that specific bidding mode, not a single rule of thumb.
- **Target CPA / Target ROAS** — the bid target. Set too aggressively
  (too low a CPA target relative to what the account can actually
  deliver), the campaign starves itself of volume rather than overspending
  — this is the most common misdiagnosis: a marketer sees low installs and
  raises budget, when the actual constraint is an unreachable CPA target.
- **Creative assets** (headlines, images, videos, HTML5) — Google auto-
  combines and auto-optimizes these; the lever is supplying enough
  *diverse* assets for the algorithm to test, not manually picking winners
  the way you would on Meta.
- **Conversion event selection & value** — which in-app event the
  campaign optimizes toward (install vs. a deeper funnel event via
  Firebase/AppsFlyer/Branch/Adjust), and the declared value if using
  Target ROAS. Optimizing toward the wrong event (e.g. install when the
  business cares about D7 retention or first purchase) is a real, common,
  and completely invisible-from-the-numbers-alone failure mode — always
  confirm which conversion action is actually selected before diagnosing
  anything else.
- **Audience signals** — hints, not restrictions (same caveat as Pmax).
  Don't expect a targeting fix here to behave like an exclusion.

## Mandatory minimum checklist — run every item, regardless of whether you already have enough findings

**Confirmed failure mode, not a hypothetical:** a real run on firstclub-v2
produced a complete, actionable CAC plan — tail-spend cuts, a reallocation
pool, creative pausing — without ever checking Ad Strength or pulling
real network-level conversion volume, even though both are documented
checks in this skill and the account is a live App Campaign account.
Nothing was *wrong* about the plan it built; it just stopped
investigating once it had found enough to build something, rather than
working through what this skill actually calls for. A plan that looks
complete is not the same as a plan built on the full checklist — the
tail-spend finding alone can't tell you whether the problem is creative
(Ad Strength) or genuine audience/bid exhaustion (network volume), and
without checking, you don't actually know which fix is correct, you've
just picked the one that happened to surface first.

**These are not optional, and "I already found something" is not a
reason to skip the rest.** For any diagnosis, audit, or optimization
task touching a live App Campaign account, run all of the following —
every one, not just whichever happens to explain what you've already
seen:

- [ ] **Ad Strength** per enabled ad, benchmarked against "Good or
  better" (`ad_group_ad.ad_strength`) — see "Ad Strength" above. **A
  Poor/weak finding is not done until it shows up as an action-plan
  line** (fix the creative, or pause it) — reporting "15/34 Good or
  better" as a standalone statistic, with nothing in the plan responding
  to it, is the same incompleteness this checklist exists to catch, just
  one level further downstream. Checking something and acting on nothing
  it found is not meaningfully different from not checking it.
- [ ] **Network-level conversion volume** (not just CTR/CPC) for every
  campaign in scope — see "CTR/CPC diagnose a network problem;
  conversion volume is what sizes the fix" above. **Postclick data in
  this codebase will never have network-level breakdown** — the
  attribution-collision fix (`docs/DECISIONS.md`, 2026-10-03/05)
  structurally collapses postclick network rows to one, permanently, not
  as a per-run surprise to rediscover. That is not a reason to mark this
  BLOCKED — it's the reason the fallback exists: Google's own
  category-filtered conversions, segmented by network
  (`segments.ad_network_type`), already confirmed composable with the
  in-app-actions query above. Use it by default. Mark this BLOCKED only
  if live Google access itself is unreachable — never because postclick
  alone doesn't have it.
- [ ] **The in-app-actions-vs-postclick gap**, at the overall campaign
  level, against the 25% threshold — see "Standing check" above. **The
  default formula is `conversions − installs`** (category-filtered,
  excluding `DOWNLOAD`) — not narrowed to one specific "canonical
  purchase action" unless you've actually found a duplicate-tracker
  problem that requires it (see "Getting from 42,619 to the correct
  3,295" above for when that narrowing is genuinely necessary). Don't
  reach for the narrower version by default; it's the exception, not the
  starting point. This check needs the same live Google access as the
  network-volume item above — if that one is BLOCKED for live-access
  reasons, say the same thing here, plainly, rather than a vaguer
  restatement that reads like a second, different problem.
- [ ] **The actual bidding goal**, via
  `campaign.selective_optimization.conversion_actions` directly — never
  inferred from the campaign name or account-level goals — see "A
  second, independent finding" above. **This rarely changes once set**,
  so it doesn't need a fresh live pull on every single analysis — if
  this skill (or recent project context) already has a confirmed finding
  for the campaign in question, that satisfies this item; only do a live
  check the first time for a given campaign, or if something in the data
  actually suggests the goal may have changed. Separately: the campaign's
  **budget, target CPA/ROAS, and bidding-strategy type are already
  stored in Supabase** without needing live access at all
  (`connector_raw_data`, `data_type='budget_bid'`, via
  `services/connectors/budgetBidHistory.js` — periodic full-state
  snapshots, not just change events) — check there before marking
  *those* specific values BLOCKED. Only the specific conversion-action
  selection itself is outside that stored vocabulary and may still need
  a live check (subject to the rarely-changes exception above).
- [ ] **`change_event`**, for any campaign whose numbers moved — see
  failure pattern #1 below. This is *meant* to sync into the same
  `budget_bid` storage as the snapshots above, via
  `services/changeHistoryService.js` — but that service has a confirmed,
  unresolved bug: it reads credentials through the legacy
  `getValidGoogleToken`/`connector_credentials` path, which is empty for
  every client (the live path is `connector_connections`, via
  `connectorsService.getConnectionIdsForPlatform`). Until that's fixed
  (a `developer` task, not something to patch around mid-analysis),
  treat `change_event` as genuinely requiring live access — don't assume
  it's sitting in Supabase just because the storage mechanism exists.

If a real time/scope constraint means one of these genuinely can't be
done, say so explicitly in the output ("Ad Strength not checked because
X") — the same discipline this skill already requires for "no playbook
exists yet" — rather than a plan that silently never mentions it was
skipped.

**Escalation — the soft version of this rule was confirmed insufficient.**
A second real run, with full live tool access the entire time (confirmed:
it ran Bash/Node commands throughout), still shipped a complete action
plan while listing 2 of these 5 items under "data pulls still needed" —
correctly *identified* as necessary, explicitly deferred to later or to
`martech-expert`, and the plan went out anyway. The third item (Ad
Strength) wasn't even listed as deferred — just absent. Having the tools
and the knowledge and still not running the check is a worse failure than
not knowing to check at all, because nothing about the output looks
incomplete — it reads as a thorough, honest plan with a footnote, which
is exactly what makes it easy to ship by mistake.

**So the rule is now structural, not advisory: the action plan section
may not be written until every checklist item is actually done.** Not
scheduled, not deferred to another agent, not listed as a future data
pull — done, in this same investigation, using the live tool access you
already have. If you notice you're about to write the action plan and a
checklist item is still open, that is the exact mid-task-roadblock moment
`performance-marketer.md`'s own ask-first discipline describes — stop and
either (a) go run the check now, since you have the tools for it, or (b)
if something genuinely prevents it (no live credential path reachable,
an explicit scope exclusion), hand back and say so plainly instead of
finishing the plan regardless.

**Required output: a literal checklist-status block, before the action
plan, every time.** Five lines, one per item, each ending in either
"— done: `<the actual finding>`" or "— BLOCKED: `<the real reason>`".
Never "— still needed" or "— for martech-expert" as a substitute for
either state; those are the exact phrasings that let this slip through
twice already, because they describe a plan to do the work rather than
the work itself or a stated blocker. A block with any BLOCKED line in it
is a visible, honest signal to the reader that the plan below is partial
— which is the whole point: make incompleteness something you have to
write down, not something that quietly happens.

## Common failure patterns, roughly in order of frequency

1. **A human changed the budget or target CPA, and the "decline" is just
   that.** Before reaching for any performance/attribution/creative
   explanation, check `change_event` (see `google-ads-api`'s "Change
   History" section) filtered to the campaign(s) in question — it names
   the exact timestamp, the exact old→new value, and the user who made
   the change. **Confirmed live (2026-09-20, supermoney-v2 RCA)**: a
   first-transaction decline that started precisely on a given date
   traced directly to a manual daily-budget cut (ACI -10%, ACE -12.2%)
   made via the Google Ads web UI ~an hour after the campaign's
   auto-optimized volume had peaked — cost and conversions fell in
   lockstep while CPA/CPI/conversion-rate stayed flat, which is the
   signature that distinguishes a delivery/budget constraint from an
   actual efficiency or creative problem (see the CPFT/CPI/conversion-rate
   check in pattern 3 below — the same "efficiency flat, volume down"
   signature applies here). This check is cheap (one `change_event` query
   scoped to the campaign, no date-range guessing) and should run *before*
   the mix-shift/creative/targeting hypotheses below, not after — it
   directly answers "was this a change someone made" rather than
   inferring it from performance data alone. Also worth scanning the same
   change window for ad-group/ad/targeting changes before concluding
   budget is the whole story — a broad change_event pull can surface
   unrelated same-day edits (new ad groups spun up and paused again,
   creative asset additions) that turn out to be net-zero or unrelated,
   which is itself useful to rule out explicitly rather than assume away.
2. **Learning-phase volatility mistaken for a real trend.** A new App
   Campaign or one that just had its budget/target changed needs days,
   not hours, to stabilize. Check the campaign's age/last-major-change
   date before reacting to a day or two of bad numbers. Per
   [Google's own guidance](https://support.google.com/google-ads/answer/14104492):
   **avoid making further changes before the first 100 conversions have
   registered** on the current settings — a real, specific threshold to
   check against, not just "give it a few days."
3. **CPA target set below what the account can deliver**, causing
   throttled volume rather than overspend — the campaign "looks fine" per
   click/impression metrics but installs stay low. Check delivery volume
   against target, not just cost efficiency.
4. **Wrong conversion event selected**, so the campaign is optimizing for
   something other than what the business actually wants — always confirm
   this explicitly, it's invisible in aggregate spend/CPI numbers.
5. **Under-budgeted relative to target CPA** (see above) — mistaken for a
   targeting/creative problem when it's a budget-vs-target math problem.
6. **Blended CPI/CAC movement driven by channel mix shift**, not this
   campaign specifically changing — always isolate this campaign's own
   trend before reacting to a blended number that includes it.
7. **Conversion tracking lag** (particularly for value-based bidding on
   deeper funnel events, which can take days to report back) confusing the
   bidding algorithm and making the campaign look erratic when it's
   actually just waiting on delayed signal.
8. **Ranking or pausing ad groups by MMP/postclick CPFT without checking
   what Google's bidding algorithm itself is actually optimizing toward.**
   Smart Bidding (tCPA/tROAS) chases whatever conversion action is marked
   `primary_for_goal` for that campaign (see `customer_conversion_goal`/
   `campaign_conversion_goal`/`conversion_action` above) — not the
   postclick source's event definition. If Branch/AppsFlyer/Adjust's
   confirmation rate against Google's own reported conversions is
   materially lower for one ad group than others in the same campaign
   (a real, live example: supermoney-v2's ACI tail ad groups confirmed
   only ~45% of what Google reported, vs. ~89% for its best ad group —
   see the change_event RCA above), that gap is a live signal to check
   *before* pausing the low-Branch-CPFT ad group as "underperforming."
   Two genuinely different explanations produce the same low-Branch-CPFT
   symptom: (a) the ad group is actually weak and Google's own numbers
   would agree, or (b) Google is crediting view-through/engaged-view
   conversions (or a broader/blended conversion action) that Branch's
   attribution model doesn't count at all — in which case the ad group
   may be delivering real value the postclick-based CPFT understates, and
   a pause decision built purely on Branch CPFT would be cutting more
   than the numbers show. **Resolve this by pulling the campaign's actual
   `primary_for_goal` conversion action(s) and comparing Google's own
   `metrics.conversions`/CPA for the specific ad group against Branch's
   FT/CPFT for the same window and same ad group** — don't infer it from
   the aggregate confirmation-rate gap alone. This is a `martech-expert`
   question, not something to resolve from performance data alone.

   **Confirmed live (2026-09-27, supermoney-v2)**: worked this exact
   check end-to-end. `campaign_conversion_goal` returned zero rows for
   both ACI and ACE — **the account-level `customer_conversion_goal` list
   does not tell you what a campaign bids toward if the campaign sets its
   own `campaign.selective_optimization.conversion_actions`**, which
   overrides it; query the campaign's own selection directly, never infer
   bidding goal from account-level goals alone. Once the right conversion
   action was isolated and compared ad-group-by-ad-group against Branch,
   the gap held up rather than disappearing: two ad groups confirmed only
   48-51% of Google's own purchase-action count (vs. 82-88% for healthier
   ad groups in the same campaign), and **the root cause was network mix,
   not attribution wrongness** — the low-confirmation ad groups spent
   72-92% of budget on YouTube, and a simple ~90%-non-YouTube /
   ~42%-YouTube confirmation-rate model reproduced the gap closely across
   every ad group checked. Two independent tracking paths (Firebase's own
   separate, non-biddable FT action, and Branch's postback into Google)
   agreed with each other on the raw count — the conversions are real,
   Branch's last-touch model simply under-credits view-driven YouTube
   traffic relative to Search/Display. **Confirming this did not overturn
   the pause case** — even measured purely on Google's own numbers, the
   flagged ad groups' CPA was still 2.1-2.4x the campaign's better ad
   groups and above target. Use this as the template: the check can either
   surface a real Branch-undercounting story (as here, where it didn't
   change the verdict) or genuinely reverse a pause decision — you don't
   know which until you've actually isolated the right conversion action
   and compared it per ad group, not in aggregate.

   **A second, independent finding from the same check, worth its own
   line**: a campaign's *name* is not proof of what it bids toward.
   supermoney-v2's ACI campaign, named `...BRFirstTran...` (implying
   first-transaction optimization), was actually bidding on **both**
   Branch's `first_open` (an install signal) and `in_app_purchase`
   simultaneously via `selective_optimization.conversion_actions` — about
   78% of its reported `metrics.conversions` were installs, not
   transactions, with no error or warning anywhere. Always check
   `campaign.selective_optimization.conversion_actions` directly rather
   than trusting the campaign name or the account-level goal list.

   **Do not treat "purchase" (or any single event) as the universally
   correct thing to bid toward, and do not present removing installs from
   a blended goal as automatically "the fix."** Finding that a campaign
   bids on an unexpected mix of events is a prompt to ask, not a verdict.
   Which event(s) should count toward bidding is a business decision
   specific to that client and that campaign's objective — a lead-gen
   client's real goal might be `SUBMIT_LEAD_FORM`, a subscription app's
   might be `SUBSCRIBE_PAID`, a lending app's might be a milestone that
   isn't "purchase" at all (KYC completion, loan disbursal), and installs
   *can* be a legitimate thing to value on their own, not just noise
   polluting a purchase-focused campaign. Two sibling campaigns disagreeing
   (one purchase-only, one blended) doesn't tell you which one is "right"
   either — either could be the accidental one. **The correct next step is
   always to ask the account owner what this specific campaign's objective
   actually is**, not to infer a target from whichever event looks most
   transaction-like, and not to assume the more common configuration in
   the account is the deliberate one.
