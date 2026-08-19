---
name: pomo-programmatic-campaign-ideas
description: Mint a Pomo programmatic API key, generate campaign ideas for a goal and audience on a named ad platform, then turn the chosen ideas into campaigns.
api: Pomo Platform API
base_url: https://api.usepomo.ai
generated: '2026-08-13'
method: generated
source: openapi/pomo-openapi.yml (harvested from https://api.usepomo.ai/openapi.json) + live 401 probes
operations:
  - list_api_keys_api_programmatic_keys_get
  - create_api_key_for_profile_api_programmatic_keys_post
  - revoke_api_key_api_programmatic_keys__api_key_id__revoke_post
  - list_api_key_usage_api_programmatic_keys__api_key_id__usage_get
  - hello_api_programmatic_v1_hello_get
  - generate_campaign_ideas_api_programmatic_v1_campaign_ideas_post
  - generate_campaigns_from_ideas_api_programmatic_v1_campaign_ideas_generate_post
---

# Generate Pomo campaign ideas programmatically

The `/api/programmatic/v1` tier is the only part of the Pomo surface designed to be called by
something other than the Pomo web app. It authenticates with an API key rather than a Clerk session
token, and the keys are self-service.

## Before you start

- Pomo publishes **no API documentation, no developer portal and no authentication guide.** Everything
  below is read from the OpenAPI contract at `https://api.usepomo.ai/openapi.json` and from live
  probes. Treat it as a reading of the contract, not as a supported integration.
- **The API key header name is not published.** `GET /api/programmatic/v1/hello`
  (`hello_api_programmatic_v1_hello_get`) answers `401 {"detail":"API key required"}` without a
  `WWW-Authenticate` challenge, so the transport location has to be confirmed from inside the product.
  Use `hello` as your credential smoke test: it is the cheapest call on the tier and exists for
  exactly this purpose.
- You need a paid Pomo account. Keys are minted per company profile from a bearer-authenticated
  session, so step 1 cannot itself be done with an API key.

## Steps

1. **Mint a key.** `POST /api/programmatic-keys`
   (`create_api_key_for_profile_api_programmatic_keys_post`, HTTP bearer). Body is
   `ApiKeyCreateRequest`: `name` (required, 1–255 chars), optional `scopes` (array of string) and
   optional `expires_at` (date-time). Response `201` / `ApiKeyCreateResponse` returns `api_key` — the
   only time the secret is returned — plus the key record.
   - **Pomo publishes no scope vocabulary.** The `scopes` field accepts arbitrary strings and there is
     no reference page listing valid values, so send it empty unless the product UI tells you otherwise.
   - Set `expires_at`. Nothing else in the contract rotates a key for you.
2. **Verify the credential.** `GET /api/programmatic/v1/hello`. A 401 here means the header name or
   the key is wrong; nothing further on the tier will work.
3. **Generate ideas.** `POST /api/programmatic/v1/campaign-ideas`
   (`generate_campaign_ideas_api_programmatic_v1_campaign_ideas_post`). Body is
   `ProgrammaticCampaignIdeasRequest`, and every constraint below is enforced by the contract:
   - `campaign_goal` — string, 1–2000 chars, required
   - `target_audience` — string, 1–2000 chars, required
   - `num_ideas` — integer, 1–30, required
   - `platform` — enum, one of `meta`, `google`, `tiktok`, `linkedin`, required
   - `social_or_ads` — enum `social` or `ads`, defaults to `social`
   The `200` response echoes your inputs and returns `target_audiences[]`, `campaign_ideas[]`,
   `organization_id`, `company_profile_id` and a `request_id`. **Keep the `request_id`** — it is the
   only correlation handle the response carries, and it matches the `X-Request-ID` response header.
4. **Turn ideas into campaigns.** `POST /api/programmatic/v1/campaign-ideas/generate`
   (`generate_campaigns_from_ideas_api_programmatic_v1_campaign_ideas_generate_post`) with
   `ProgrammaticCampaignGenerationRequest`. This is the mutating step — it creates work inside the
   company profile the key belongs to.
5. **Watch spend and revoke.** `GET /api/programmatic-keys/{api_key_id}/usage`
   (`list_api_key_usage_api_programmatic_keys__api_key_id__usage_get`) returns per-key usage events.
   `POST /api/programmatic-keys/{api_key_id}/revoke` kills a key immediately.

## Rules an agent must follow

- **Rate limit.** Every response carries `X-RateLimit-Limit` (observed: 60),
  `X-RateLimit-Remaining` and `X-RateLimit-Reset` (unix epoch seconds). The window length is not
  published, `429` is declared on none of the 994 operations, and no `Retry-After` is returned — so
  read `X-RateLimit-Remaining` on every response and back off before you hit zero rather than after.
- **No idempotency key on this tier.** `idempotency_key` exists on two unrelated request schemas
  (`B2BSaveCandidateRequest` and the agentic conversation request); the programmatic endpoints have
  none. Step 4 is not safe to blind-retry — re-check state before retrying a generation call.
- **Errors.** `422` returns `HTTPValidationError`: `detail[]` of `{loc, msg, type}`, where `loc` is the
  path to the offending field. Everything else returns `{"detail": "<message>"}`. There is no
  `application/problem+json` and no error-code registry.
- **Generation costs credits.** Ad and generation work draws on a credit ledger
  (`/api/payment/credits/balance`, `.../hold`, `.../capture`, `.../release`). Check the balance before
  a large `num_ideas` run.
