---
name: Trigger a Pinkfish workflow and get results
description: >-
  Execute a published Pinkfish workflow through its API trigger, either waiting
  synchronously for results or firing asynchronously and polling the run to
  completion.
api: openapi/pinkfish-triggers-openapi.yml
operations: [triggerWorkflow, getRunStatus, getRunResults]
---

# Trigger a Pinkfish workflow and get results

Use the Pinkfish Triggers API to run a workflow you have already created and
published, then retrieve its output.

## Prerequisites

- A published workflow with an API trigger configured (note its `triggerId`).
- An API key. Generate one in the platform **Library** section or during
  trigger setup. Pass it in the `X-API-KEY` request header.

## Authentication

Send the API key in the header on every call:

```
X-API-KEY: YOUR_API_KEY
```

For third-party callers that cannot set headers, use the webhook variant which
embeds the key in the path (`triggerWorkflowViaWebhook`):
`/ext/webhook/{apiKey}/triggers/{triggerId}`.

## Option A — synchronous (wait for the result)

1. Call `triggerWorkflow` — `POST /ext/triggers/{triggerId}` — with header
   `x-api-wait: true` and your JSON input body.
2. If the workflow finishes within 60 seconds, the `200` response body is the
   full `RunResult` (per-step `results[]`, `status: COMPLETE`).
3. If it is still running when the window closes, you get `202 Accepted` with a
   `Retry-After` header and a handle (`runId`, `statusUrl`, `resultsUrl`) — fall
   through to the polling steps below.

## Option B — asynchronous (fire and poll)

1. Call `triggerWorkflow` with `x-api-wait: false` (or omit the header). The
   body is `null`; read the run id from the `X-Pf-Run-Id` response header and
   the automation id from `X-Pf-Automation-Id`.
2. Poll `getRunStatus` — `GET /ext/webhook/{apiKey}/runs/{automationId}/{runId}/status`
   every 2-5 seconds. Polling always uses the webhook-style URL even if you
   triggered with the header endpoint.
3. Keep polling while `status` is `PENDING`, `RUNNING`, or `PAUSED` (PAUSED is a
   durable wait/approval, not terminal). Stop on a terminal status: `COMPLETE`,
   `FAILED`, or `TIMEOUT`.
4. When `COMPLETE`, call `getRunResults` —
   `GET /ext/webhook/{apiKey}/runs/{automationId}/{runId}/results` — to fetch the
   full `RunResult`, including signed `resultUrls` for output files.

## Conventions and error handling

- Content types: `application/json`, `application/x-www-form-urlencoded`, or
  `multipart/form-data` (file attachments, API triggers only — max 25MB/file,
  100MB total, 20 files; signed URLs valid 24h).
- Errors return `{ "error": ..., "message": ... }`. A `401` means a missing or
  invalid API key; a `400` means a malformed request.
- No idempotency-key contract is documented — do not assume safe automatic
  retries of a trigger; prefer polling an existing run over re-triggering.

## Related

- MCP: the embedded Pinkfish Sidekick server exposes `workflow_run`,
  `workflow_run_status`, and `workflow_results` tools that handle polling
  automatically (`mcp/pinkfish-mcp.yml`).
