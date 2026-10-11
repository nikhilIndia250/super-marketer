---
name: meta-ads
description: Use when working with Meta Ads campaign structure, objectives, ad sets, audiences, bidding, or diagnosing Facebook and Instagram ad performance. Platform concepts, not API code.
---

# Meta Ads Platform Knowledge

## Campaign Structure (hierarchy)
Campaign → Ad Set → Ad

## Campaign Objectives (Meta's current structure)
- **Awareness** — Brand Awareness, Reach, Video Views
- **Traffic** — link clicks, landing page views
- **Engagement** — post engagement, video views, messages, conversions
- **Leads** — instant forms, Messenger, calls, website leads
- **App Promotion** — app installs, app events (replaces old Mobile App Install objective)
- **Sales** — conversions, catalogue sales, Messenger, calls (most common for e-commerce)

## Ad Set Level (key controls)
- **Budget** — daily or lifetime (set at ad set level, or campaign level with CBO)
- **Schedule** — start/end date, dayparting
- **Audience** — Core (interest/demo), Custom Audience, Lookalike Audience
- **Placements** — Advantage+ Placements (automatic) or Manual (Facebook Feed, Instagram Feed, Stories, Reels, Audience Network, Messenger, etc.)
- **Optimisation event** — what Meta optimises for (Purchase, Add to Cart, ViewContent, Install, etc.)
- **Bid strategy** — Highest Volume, Cost Per Result Goal, ROAS Goal, Bid Cap, Minimum ROAS

## Ad Types
- Image ad
- Video ad
- Carousel ad (multiple images/videos)
- Collection ad (cover + product grid)
- Instant Experience (full-screen mobile)
- Dynamic ads (catalogue-based, personalised per user)
- Lead form ads
- Story/Reel ads

## Advantage+ (AI-powered products)
- **Advantage+ Shopping Campaigns (ASC)** — fully automated shopping, similar to Pmax. Best performers for e-commerce.
- **Advantage+ Audience** — AI expands beyond defined audience
- **Advantage+ Creative** — AI optimises creative elements (brightness, text, music)
- **Advantage+ Placements** — AI picks best placements automatically

## Key Metrics
- **Impressions** — times ad was shown
- **Reach** — unique accounts that saw ad
- **Frequency** — impressions / reach
- **Clicks (All)** — all clicks including likes, comments
- **Link Clicks** — clicks to destination URL
- **CTR (Link Click-Through Rate)** — link clicks / impressions
- **CPC (Cost per Link Click)**
- **CPM** — cost per 1000 impressions
- **Results** — campaign objective metric (purchases, installs, etc.)
- **Cost per Result** — cost / results
- **ROAS (Return on Ad Spend)** — purchase value / spend
- **Purchase Value** — total revenue attributed
- **Add to Cart** — ATC events
- **Initiate Checkout**
- **Video Views** — 3-second, ThruPlay (15s or completed)
- **Video View Rate** — views / impressions

## Attribution Windows
- Click: 1-day, 7-day (default)
- View: 1-day (optional)
- Attribution setting affects reported conversions significantly
- iOS14+ limits to 1-day click attribution for iOS app campaigns

## Pixel & Conversions API
- **Meta Pixel** — browser-side tracking (affected by iOS14/ad blockers)
- **Conversions API (CAPI)** — server-side tracking (more reliable, complements Pixel)
- **Event Match Quality** — score for how well events are matched to users
- Standard events: Purchase, AddToCart, ViewContent, InitiateCheckout, Lead, CompleteRegistration, Search, Subscribe

## Audiences
- **Core audiences** — demographics, interests, behaviours
- **Custom audiences** — website visitors (Pixel), customer list (email/phone), app activity, video viewers, Instagram/Facebook engagers
- **Lookalike audiences** — 1-10% similarity to source audience
- **Advantage+ Audience** — AI-driven, no manual constraints

## Campaign Budget Optimisation (CBO)
- Budget set at campaign level, Meta distributes across ad sets
- Good for scaling winners automatically
- ABO (Ad Set Budget Optimisation) — budget set per ad set, more control

## Key Dimensions for Reporting
- Age (13-17, 18-24, 25-34, 35-44, 45-54, 55-64, 65+)
- Gender (Male, Female, Unknown)
- Country / Region / City / DMA
- Placement (Facebook Feed, Instagram Feed, Stories, Reels, Audience Network, Messenger)
- Platform (Facebook, Instagram, Audience Network, Messenger)
- Device (Mobile, Desktop)
- Publisher platform
- Time (Day, Week, Month, Hour of Day, Day of Week)

## iOS14+ Impact
- ATT (App Tracking Transparency) — users must opt-in to tracking
- Limited signal for iOS app installs and in-app events
- Aggregated Event Measurement — max 8 events per domain prioritised
- Conversion modelling fills gaps in opted-out users
- SKAdNetwork (SKAN) attribution for iOS app campaigns — delayed and aggregated
