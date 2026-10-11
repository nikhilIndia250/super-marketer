---
name: google-ads
description: Use when working with Google Ads campaign structure, campaign types (Search, PMax, UAC, Demand Gen), bidding strategies, or diagnosing Google Ads performance. Platform concepts, not API code.
---

# Google Ads Platform Knowledge

## Campaign Structure (hierarchy)
Account → Campaign → Ad Group → Ad → Keyword/Audience

## Campaign Types
- **Search** — text ads triggered by keywords
- **Performance Max (Pmax)** — fully automated, runs across all Google channels (Search, Display, YouTube, Gmail, Maps, Discover). Single campaign with asset groups. No ad group level.
- **Display** — banner ads across Google Display Network
- **Shopping / Standard Shopping** — product listing ads, feed-based
- **App Campaigns** — promote app installs or in-app actions (iOS/Android). No ad groups, no keywords — Google auto-optimises.
- **Video (YouTube)** — skippable in-stream, non-skippable, bumper, responsive video
- **Demand Gen** — visual ads on YouTube, Discover, Gmail. Successor to Discovery campaigns.
- **Smart campaigns** — simplified automated campaigns (SMB-focused)

## Bidding Strategies
- **Manual CPC** — set bids manually
- **Target CPA (tCPA)** — automated, optimise for cost per acquisition
- **Target ROAS (tROAS)** — automated, optimise for return on ad spend
- **Maximize Conversions** — spend budget to get most conversions
- **Maximize Conversion Value** — spend budget to get highest value
- **Enhanced CPC (eCPC)** — manual with automated adjustments
- **Target Impression Share** — for brand visibility

## Key Metrics
- **Impressions** — times ad was shown
- **Clicks** — times ad was clicked
- **CTR** — clicks / impressions
- **CPC** — cost per click (avg)
- **CPM** — cost per 1000 impressions
- **Conversions** — completed goal actions
- **Conversion Rate** — conversions / clicks
- **CPA** — cost per conversion
- **ROAS** — revenue / cost
- **Search Impression Share** — impressions / eligible impressions
- **Quality Score** — 1-10 score (Expected CTR + Ad Relevance + Landing Page Experience)
- **View-through conversions** — converted after seeing (not clicking) an ad

## Pmax Specifics
- Asset groups replace ad groups
- Assets: headlines (up to 15), descriptions (up to 4), images, logos, videos, sitelinks
- Audience signals — hints not restrictions
- No search term level reporting (limited)
- Insights tab for asset group performance
- Budget = campaign level only
- Cannot exclude keywords at ad group level (limited campaign-level negatives)

## App Campaign Specifics
- Subtypes: App Installs, App Engagement, App Pre-registration
- Bidding: Target CPA for installs, Target CPA/ROAS for engagement
- Assets: text, images, videos, HTML5
- Deep links for engagement campaigns
- In-app event optimisation via linked Firebase/third-party (AppsFlyer/Branch/Adjust)

## Account Structure Concepts
- **MCC (My Client Centre)** — manager account containing multiple client accounts
- **Customer ID** — 10-digit ID (xxx-xxx-xxxx format) identifying each account
- **Conversion actions** — defined goals (purchase, signup, app install, etc.)
- **Shared budgets** — one budget shared across multiple campaigns
- **Labels** — tags for organising and filtering campaigns/ad groups/ads
- **Negative keywords** — at campaign or ad group level
- **Audience lists** — remarketing, similar audiences, Customer Match, in-market

## Attribution Models (in Google Ads)
- Data-driven (default, recommended)
- Last click
- First click
- Linear
- Time decay
- Position-based

## Common Segmentation Dimensions
- Device (Mobile, Desktop, Tablet)
- Network (Google Search, Search Partners, Display, YouTube)
- Day of week / Hour of day
- Geography (country, region, city)
- Audience segment
- Ad format
