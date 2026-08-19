---
name: pomo-agentic-jobs
description: Start a Pomo agentic job, stream its progress over Server-Sent Events, poll status as a fallback, and cancel it — the long-running work pattern the whole platform uses.
api: Pomo Platform API
base_url: https://api.usepomo.ai
generated: '2026-08-13'
method: generated
source: openapi/pomo-openapi.yml (harvested from https://api.usepomo.ai/openapi.json) + https://api.usepomo.ai/health/events
operations:
  - start_async_job_api_chat_agentic_async_post
  - list_jobs_api_chat_agentic_jobs_get
  - get_job_status_api_chat_agentic_jobs__job_id__get
  - stream_job_progress_api_chat_agentic_jobs__job_id__stream_get
  - cancel_job_api_chat_agentic_jobs__job_id__cancel_post
  - stream_team_activity_api_agentic_teams__team_id__stream_get
  - stream_user_conversation_events_api_agentic_user_conversation_stream_get
  - stream_all_workflow_events_api_brand_workflow_sse_stream_get
  - stream_brand_workflow_events_api_brand_workflow_sse_stream__process_id__get
  - get_workflow_state_api_brand_workflow_sse_state__process_id__get
  - get_active_workflow_api_brand_workflow_sse_active_get
---

# Run long work on Pomo and watch it happen

Pomo's generative work is asynchronous. The pattern is consistent enough to learn once: a `POST`
starts a job and returns an id, a `GET .../stream` gives you Server-Sent Events, and a `GET` on the
job gives you a snapshot. Bearer auth throughout.

## Steps

1. **Start the job.** `POST /api/chat/agentic/async` (`start_async_job_api_chat_agentic_async_post`)
   returns a job id.
2. **Subscribe.** `GET /api/chat/agentic/jobs/{job_id}/stream`
   (`stream_job_progress_api_chat_agentic_jobs__job_id__stream_get`) is a Server-Sent Event stream.
   - **The contract does not say so.** None of the twelve streaming operations declares
     `text/event-stream`, so a generated client will treat this as an ordinary JSON `GET` and hang.
     Set `Accept: text/event-stream` and read it as a stream yourself.
   - **Resume with `Last-Event-ID`.** The transport accepts it; use it after a disconnect instead of
     restarting the job.
3. **Poll as a fallback.** `GET /api/chat/agentic/jobs/{job_id}`
   (`get_job_status_api_chat_agentic_jobs__job_id__get`) is the snapshot; `GET /api/chat/agentic/jobs`
   lists jobs. Poll only if the stream drops — the rate limit is 60 per window and a tight poll loop
   will burn it.
4. **Cancel work you no longer need.** `POST /api/chat/agentic/jobs/{job_id}/cancel`. Generative work
   draws credits; abandoning a job without cancelling it does not stop the spend.
5. **Reconcile after a drop.** For brand workflows, `GET /api/brand-workflow/sse/state/{process_id}`
   (`get_workflow_state_api_brand_workflow_sse_state__process_id__get`) returns point-in-time state,
   and `GET /api/brand-workflow/sse/active` tells you what is currently running. Use these instead of
   replaying a stream from the start.

## The other streams

| Stream | Scope |
|---|---|
| `GET /api/agentic/teams/{team_id}/stream` | live agent-team activity |
| `GET /api/agentic/user/conversation-stream` | all conversation events for the caller |
| `GET /api/brand-workflow/sse/stream` | every brand-workflow event for the caller |
| `GET /api/brand-workflow/sse/stream/{process_id}` | one brand-generation process |

## Rules an agent must follow

- **Idempotency, where it exists.** The agentic conversation request carries an optional
  `idempotency_key` (string, 16–200 chars). Send one on message submission — it is the only
  idempotency guarantee available on this surface, and it is a body field, not a header.
- **Confirmation before writes.** The MCP-shaped tool broker (`/api/mcp/tools/execute`) takes a
  `confirmation_token` described as "Confirmation token for write operations". Where a flow offers
  that gate, use it: this platform launches paid ad campaigns.
- **There is no webhook.** Nothing in this 994-operation contract lets a customer register a callback
  URL. SSE plus polling is the whole story — do not look for a webhook subscription endpoint.
- **Capacity is public.** `GET /health/events` is unauthenticated and reports live SSE connection
  counts against a 1000-connection ceiling. Useful for diagnosing a refused stream.
- **Errors.** `422` -> `HTTPValidationError`; otherwise `{"detail": "<message>"}`. `429` is undeclared
  but the `X-RateLimit-*` headers are real — read them.
