---
name: pomo-competitor-tracking
description: Set up tracked competitors on a Pomo company profile, trigger and poll a scan, then read insights and captured Facebook ad creative.
api: Pomo Platform API
base_url: https://api.usepomo.ai
generated: '2026-08-13'
method: generated
source: openapi/pomo-openapi.yml (harvested from https://api.usepomo.ai/openapi.json)
operations:
  - get_company_profiles_api_company_profile__get
  - setup_competitors_api_competitor_tracking_setup_post
  - list_competitors_api_competitor_tracking_list_get
  - list_competitor_statuses_api_competitor_tracking_status_get
  - scan_competitor_api_competitor_tracking_scan__competitor_id__post
  - get_competitor_scan_status_api_competitor_tracking__competitor_id__scan_status_get
  - get_competitor_details_api_competitor_tracking__competitor_id__get
  - get_competitor_details_batch_api_competitor_tracking_details_batch_get
  - get_competitor_insights_api_competitor_tracking__competitor_id__insights_get
  - get_competitor_facebook_ads_api_competitor_tracking__competitor_id__facebook_ads_get
  - update_competitor_handles_api_competitor_tracking__competitor_id__update_handles_post
  - archive_competitor_api_competitor_tracking__competitor_id__archive_post
  - restore_competitor_api_competitor_tracking__competitor_id__restore_post
  - get_company_profile_limits_api_limits_company_profile__profile_id__get
---

# Track competitors in Pomo

Competitor tracking is one of Pomo's marquee flows: 23 paths under `/api/competitor-tracking`,
scoped to a company profile. All of it requires an HTTP bearer token (Clerk-issued session).

## Establish context first

Almost nothing in this API works without a company profile id.

1. `GET /api/company-profile/` (`get_company_profiles_api_company_profile__get`) to resolve the
   profiles the caller can see.
2. `GET /api/limits/company-profile/{profile_id}`
   (`get_company_profile_limits_api_limits_company_profile__profile_id__get`) to read the plan quota
   before you add anything. Competitor slots are plan-limited — the published tiers allow 5
   competitors per profile — and adding past the limit is a wasted call.
3. Carry `X-Organization-Id` on requests that accept it (59 operations do), and pass
   `company_profile_id` as the query parameter everywhere else. This is the single most common cause
   of an empty result set that looks like a bug.

## Steps

1. **Register competitors.** `POST /api/competitor-tracking/setup`
   (`setup_competitors_api_competitor_tracking_setup_post`).
2. **List what is being tracked.** `GET /api/competitor-tracking/list`
   (`list_competitors_api_competitor_tracking_list_get`), and
   `GET /api/competitor-tracking/status` (`list_competitor_statuses_api_competitor_tracking_status_get`)
   for the per-competitor state in one call.
3. **Fix handles before you scan.** `POST /api/competitor-tracking/{competitor_id}/update-handles`
   (`update_competitor_handles_api_competitor_tracking__competitor_id__update_handles_post`). Social
   handle resolution is what feeds ad capture; a wrong handle yields a clean-looking empty scan.
   `GET /api/competitor-tracking/{competitor_id}/brand-facebook-official-url` helps confirm the page.
4. **Scan.** `POST /api/competitor-tracking/scan/{competitor_id}`
   (`scan_competitor_api_competitor_tracking_scan__competitor_id__post`). This is asynchronous.
5. **Poll the scan.** `GET /api/competitor-tracking/{competitor_id}/scan-status`
   (`get_competitor_scan_status_api_competitor_tracking__competitor_id__scan_status_get`) until it
   settles. There is no webhook and no SSE stream for competitor scans — polling is the only option
   this contract offers for this flow.
6. **Read the result.**
   - `GET /api/competitor-tracking/{competitor_id}/insights` for the analysis.
   - `GET /api/competitor-tracking/{competitor_id}/facebook-ads` for captured ad creative.
   - `GET /api/competitor-tracking/details-batch` when you need several competitors at once — prefer
     it over N single `GET /api/competitor-tracking/{competitor_id}` calls, because the rate limit
     observed on this API is 60 per window.
7. **Retire, do not delete.** `POST /api/competitor-tracking/{competitor_id}/archive` is reversible
   via `.../restore`. `DELETE /api/competitor-tracking/{competitor_id}` is not. Archive frees the
   plan slot without losing history.

## Rules an agent must follow

- **Rate limit.** `X-RateLimit-Limit: 60` with `X-RateLimit-Remaining` and `X-RateLimit-Reset` on
  every response, including 401s and 404s. Nothing is documented and `429` is declared on no
  operation; read the remaining count and pace yourself. Batch endpoints exist precisely for this.
- **Errors.** `422` -> `HTTPValidationError` (`detail[]` of `{loc, msg, type}`); everything else
  `{"detail": "<message>"}`. Not RFC 9457.
- **No idempotency.** None of these operations accepts an idempotency key. `setup` and `scan` are
  not safe to blind-retry — re-read `list` or `scan-status` first.
- **Pagination.** The competitor ad list returns `CompetitorAdListResponse` with `page`, `limit`,
  `total` and `total_pages` — page/limit style. Other areas of this API use offset/limit or cursors
  instead; do not assume one style carries across surfaces.
