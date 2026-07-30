---
name: macroscope-weekly-changes-digest
description: Trigger Macroscope's agent to summarize recent codebase changes and retrieve the answer, either delivered to Slack or by polling.
api: Macroscope Agent Webhook API
source: https://docs.macroscope.com/api.md
operations:
  - queryAgentWebhookTrigger
  - pollAgentJob
generated: '2026-07-20'
method: generated
---

# Macroscope: Weekly Changes Digest

Ask Macroscope's agent what changed recently and route the answer to Slack, or
poll for it inline. Grounded in the two real operations of the agent webhook
API (`queryAgentWebhookTrigger`, `pollAgentJob`).

## Prerequisites

- Status must be configured (Product Overview + Sprint Cadence) and the backfill
  finished, or the API will not run.
- A webhook API key from **Settings -> Connections -> Webhooks** (shown once).
- Your **Trigger URL** with your `{workspaceType}` and `{workspaceId}`.

## Steps

1. **Trigger the agent** (`queryAgentWebhookTrigger`). POST to the trigger URL
   with the `X-Webhook-Secret` header and a JSON body containing your `query`.
   To deliver straight to Slack, include
   `responseDestination: { "slackChannelId": "C0123456789" }`.

   ```bash
   curl -X POST \
     "https://hooks.macroscope.com/api/v1/workspaces/{workspaceType}/{workspaceId}/query-agent-webhook-trigger" \
     -H "Content-Type: application/json" \
     -H "X-Webhook-Secret: $MACROSCOPE_API_KEY" \
     -d '{ "query": "What were the main changes this week?", "timezone": "America/Los_Angeles" }'
   ```

   You get `202 Accepted` with `{ jobToken, pollUrl, workflowId }`
   (`workflowId` is deprecated).

2. **Poll for the result** (`pollAgentJob`) — only if you omitted a
   `responseDestination`. Send the `jobToken` as a bearer token to the
   `pollUrl`:

   ```bash
   curl -i "$POLL_URL" -H "Authorization: Bearer $JOB_TOKEN"
   ```

   - `202` + `{ "status": "running" }` -> wait the `Retry-After` seconds, poll again.
   - `200` + `{ "status": "completed", "response": "..." }` -> done; use `response`.
   - `200` + `{ "status": "failed" }` -> the job failed; re-trigger.

## Rules and gotchas

- The job token lives ~1 hour; a `403` while polling means it expired — re-trigger.
- External `webhookUrl` destinations must be HTTPS and allowlisted by a GitHub
  org admin (Settings -> Connections -> Webhooks -> Allowed External URLs).
- Errors are plain HTTP codes (400 missing query, 401 bad secret, 403 wrong
  workspace / not allowlisted). See `errors/macroscope-problem-types.yml`.
