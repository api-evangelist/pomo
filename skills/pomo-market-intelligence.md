---
name: pomo-market-intelligence
description: Generate and read a Pomo market intelligence report, pull the multi-platform trend bundle, and work the social listening and prospecting surfaces.
api: Pomo Platform API
base_url: https://api.usepomo.ai
generated: '2026-08-13'
method: generated
source: openapi/pomo-openapi.yml (harvested from https://api.usepomo.ai/openapi.json)
operations:
  - generate_market_intelligence_api_market_intelligence_intelligence_generate_post
  - get_market_intelligence_api_market_intelligence_intelligence_get
  - get_intelligence_report_dates_api_market_intelligence_intelligence_report_dates_get
  - get_intelligence_item_api_market_intelligence_intelligence__intelligence_id__get
  - update_intelligence_item_api_market_intelligence_intelligence__intelligence_id__patch
  - refresh_market_intelligence_api_market_intelligence_intelligence_refresh_post
  - get_trends_bundle_api_market_intelligence_trends_bundle_get
  - get_google_trends_analysis_api_market_intelligence_google_trends_get
  - get_tiktok_trends_analysis_api_market_intelligence_tiktok_trends_get
  - get_instagram_trends_analysis_api_market_intelligence_instagram_trends_get
  - get_youtube_trends_analysis_api_market_intelligence_youtube_trends_get
  - get_amazon_trends_analysis_api_market_intelligence_amazon_trends_get
  - get_yelp_trends_analysis_api_market_intelligence_yelp_trends_get
  - get_ads_winners_api_market_intelligence_ads_winners_get
  - get_competitive_intelligence_api_market_intelligence_competitive_intelligence_get
  - get_social_listening_dashboard_api_market_intelligence_social_listening_dashboard_get
  - get_social_listening_mentions_api_market_intelligence_social_listening_mentions_get
  - update_social_listening_settings_api_market_intelligence_social_listening_settings_put
  - get_social_prospecting_api_market_intelligence_social_prospecting_get
  - export_social_prospecting_api_market_intelligence_social_prospecting_export_get
  - analyze_market_evolution_api_market_intelligence_intelligence_evolution_analyze_post
  - get_evolution_history_api_market_intelligence_intelligence_evolution_history_get
---

# Pull market intelligence out of Pomo

`/api/market-intelligence` is the "always-on market intelligence" the product is sold on, exposed as
27 paths. Bearer auth throughout; everything is scoped to a company profile.

## Steps

1. **Find out what already exists before you generate anything.**
   `GET /api/market-intelligence/intelligence/report-dates`
   (`get_intelligence_report_dates_api_market_intelligence_intelligence_report_dates_get`) lists the
   dates that already have reports. Generation is expensive and draws credits — do not regenerate a
   report that exists.
2. **Read the current report.** `GET /api/market-intelligence/intelligence`
   (`get_market_intelligence_api_market_intelligence_intelligence_get`). Many endpoints in this area
   accept `force_refresh` (22 operations across the API do) — leave it off unless you specifically
   need to bust a cache, because it turns a read into work.
3. **Generate or refresh only when needed.**
   `POST /api/market-intelligence/intelligence/generate` for a new report;
   `POST /api/market-intelligence/intelligence/refresh` to update the existing one.
4. **Pull trends.** `GET /api/market-intelligence/trends-bundle`
   (`get_trends_bundle_api_market_intelligence_trends_bundle_get`) is the aggregate. The per-source
   endpoints — Google, TikTok, Instagram, YouTube, Amazon, Yelp — exist when you need one source
   only. Prefer the bundle: the observed rate limit is 60 per window and six calls to do one job is
   a poor trade.
5. **Read the competitive layer.** `GET /api/market-intelligence/competitive-intelligence` and
   `GET /api/market-intelligence/ads-winners` (top-performing ad creative in the category).
6. **Social listening.** `GET .../social-listening-dashboard` for the roll-up,
   `GET .../social-listening-mentions` for the raw mentions, and
   `PUT .../social-listening-settings` to change what is monitored. Settings are a `PUT` —
   read the current settings first, or you will overwrite them.
7. **Social prospecting.** `GET .../social-prospecting` returns qualified leads;
   `GET .../social-prospecting/export` exports them; and
   `PATCH .../social-prospecting-candidates/{candidate_id}` moves one candidate's status. Note this
   surface is plan-gated — social prospecting is an Accelerate-and-above feature.
8. **Track how the market moved.** `POST .../intelligence/evolution/analyze` then
   `GET .../intelligence/evolution/history`.
9. **Suppress noise.** `PUT /api/market-intelligence/trend-suppressions` mutes a trend that keeps
   surfacing and should not.

## Rules an agent must follow

- **This is generative work with a cost.** Generate/refresh/analyze draw on the organization's credit
  ledger. Check `GET /api/payment/credits/balance` before a batch run and prefer the cached read.
- **Date windows.** `start_date` / `end_date` (38 operations each) and `days` (18) are the standard
  window parameters across this area.
- **Rate limit.** 60 per window, signalled by `X-RateLimit-*` on every response, undocumented and
  with no `Retry-After`. Aggregate endpoints exist — use them.
- **Errors.** `422` -> `HTTPValidationError`; otherwise `{"detail": "<message>"}`.
- **Correlate with `X-Request-ID`.** Every response carries it (and an identical `X-Trace-ID`). It is
  the only support handle this API offers, and there is no support channel for the API itself.
