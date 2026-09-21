---
name: crawlbrulee-scrape-async
description: use when a crawlbrulee scrape shouldn't hold the connection open — submitting a background job, polling its status, fetching the result later, or being notified by a signed scrape.complete webhook instead of polling. covers the job lifecycle, webhook payloads, signature verification, and secret rotation.
license: Apache-2.0
metadata:
  author: crawlbrulee
  homepage: https://crawlbrulee.com
---

# 🍮 crawlbrulee async scrape & webhooks

an async scrape takes the same request body as a sync one, but returns a `job_id` immediately instead of making you wait. you then either poll for the result or let a webhook tell you it's ready.

new to crawlbrulee? read the **crawlbrulee** skill first. the request body and the result shape are the sync contract — see **crawlbrulee-scrape**.

## when to go async

- **slow pages** — heavy JavaScript rendering, or full-page screenshots of very long pages.
- **batches** — you have many urls and don't want a connection open per url.
- **you'd rather be called than poll** — attach a webhook and react on completion.

for a single quick fetch, stay synchronous. it's simpler and you get the content in one call.

## the lifecycle

three steps: submit → poll status → fetch result.

```bash
# 1. submit — same body as POST /api/scrape
curl -sX POST https://api.crawlbrulee.com/api/scrape/async \
  -H "Authorization: Bearer $CRAWLBRULEE_API_KEY" -H "Content-Type: application/json" \
  -d '{"url":"https://example.com","extract":{"markdown":true}}'
# → 202 { "job_id": "683a1f2b4c5d6e7f8a9b0c1d" }

# 2. poll until terminal
curl -s https://api.crawlbrulee.com/api/scrape/status/683a1f2b4c5d6e7f8a9b0c1d \
  -H "Authorization: Bearer $CRAWLBRULEE_API_KEY"
# → { "job_id": "…", "status": "running", "created_at": "2026-07-13T10:00:00.000Z" }

# 3. fetch the result — same shape as a sync scrape
curl -s https://api.crawlbrulee.com/api/scrape/result/683a1f2b4c5d6e7f8a9b0c1d \
  -H "Authorization: Bearer $CRAWLBRULEE_API_KEY"
```

**job states: `pending` → `running` → `done` | `failed`.** the status body is snake_case throughout (`job_id`, `created_at`).

- once `done`, the **status** body also carries `response_meta.usage` — so you can see what a job charged without fetching the whole result.
- on `failed`, `error` explains what went wrong.
- **`result` errors if the job isn't finished** — poll `status` first. it also `404`s for an unknown job, or one whose result has aged out.

results don't live forever; see [async scrape](https://crawlbrulee.com/docs/scrape/async) for the current retention window. poll on a sensible interval — a couple of seconds is plenty. every client ships a helper that wraps this loop (`waitForScrape` / `wait_for_scrape` / `crawlbrulee scrape wait`), so prefer that over hand-rolling it.

## webhooks: get called instead of polling

attach a `webhook` to the submit body and we deliver a single signed `scrape.complete` `POST` when the job reaches a terminal state.

```jsonc
{
  "url": "https://example.com",
  "extract": { "markdown": true },
  "webhook": {
    "url": "https://hooks.example.com/crawlbrulee",
    "metadata": { "order": "abc", "attempt": 2 }
  }
}
```

- **`webhook.url`** — the endpoint that receives the `POST`. https is required in production.
- **`webhook.metadata`** — an opaque object echoed back verbatim in the delivery. use it to correlate a job with your own records instead of keeping a `job_id` mapping.

webhooks are **async-only** — the sync `/api/scrape` response *is* your notification, so it doesn't accept a `webhook`, and sending one is a validation error.

the destination is per job; the **signing secret is per organization** and configured once in the dashboard under Account → Webhooks. there's no per-request secret.

### the payload

```jsonc
{
  "event_id": "…",
  "timestamp": "…",
  "event": "scrape.complete",
  "data": {
    "job_id": "…",
    "status": "success",
    "url": "https://example.com",
    "completed_at": "…",
    "metadata": { "order": "abc", "attempt": 2 },
    "response_meta": { "usage": { "credits": 1, "engine": "http", "proxy": "basic", "screenshot_slices": 0 } }
  }
}
```

**the delivery is a pointer, not the content** — it never carries the scraped page. read `data.job_id` and fetch the result from `GET /api/scrape/result/{job_id}`.

**`data.status` uses a different vocabulary from job status: `success` | `failed` | `cancelled`** — not `done`. this trips people up. `response_meta` is present only on `success`; `error` only on `failed`.

deliveries are at-least-once and we retry a failed delivery a few times before giving up. a delivery counts as accepted only on a `2xx` — a redirect is treated as a failure, so point the webhook straight at the final url. **de-duplicate on `event_id`** (also sent as the `X-Cwbl-Event-Id` header), which stays stable across retries.

### verifying the signature

every delivery is signed. **verify before you parse or trust the body.**

- **`X-Cwbl-Signature`** — always present, format `t=<unix_seconds>,v1=<hex>`.
- the signed payload is `` `{timestamp}.{raw_body}` ``, hmac-sha256 with your secret, lowercase hex.
- **use the raw request bytes.** re-serialized json will not match.
- compare in constant time, and reject deliveries whose timestamp is outside a tolerance window (replay protection) — the sdk helpers default to a sensible one.

every first-party client ships a verifier that returns a result rather than throwing, because a forged delivery is normal control flow, not an exception:

```ts
// js/ts — note it's async
import { verifyWebhookSignature } from '@crawlbrulee/sdk'

const result = await verifyWebhookSignature({
  payload: rawBody,                                  // raw string or bytes, never re-serialized
  headers: req.headers,
  secret: process.env.CRAWLBRULEE_WEBHOOK_SECRET!,
})
if (!result.verified) return res.status(400).end()   // result.reason says why
```

```python
# python — keyword-only
from crawlbrulee import verify_webhook_signature

result = verify_webhook_signature(
    payload=raw,                                     # raw bytes from the request
    headers=request.headers,
    secret=WEBHOOK_SECRET,
)
if not result.verified:
    return Response(status_code=400)                 # result.reason says why
```

failure reasons are `missing_signature`, `malformed_signature`, `timestamp_out_of_tolerance`, and `signature_mismatch`.

then hand the verified body to the client to fetch the page:

```ts
const page = await cb.fetchScrapeResultFromWebhook(webhook)   // throws on failed / cancelled jobs
```

### secret rotation

during a rotation grace window we send a **second** `X-Cwbl-Signature-Rotated` header signed with the previous secret. keep passing your current secret — the verifiers try the primary header first, then the rotated one, and report which matched (`signedWith` / `signed_with`, either `primary` or `rotated`). verification keeps working whether or not you've picked up the new secret yet.

the previous secret stays valid until you explicitly revoke it or rotate again, so there's no deadline to race.

## see also

- the request body and result shape: **crawlbrulee-scrape** · shared contract: **crawlbrulee-api**
- helpers per interface: **crawlbrulee-cli** (`--async`, `--wait`, `--webhook-url`), **crawlbrulee-sdk-js**, **crawlbrulee-sdk-python**, **crawlbrulee-mcp**
- docs: [async scrape](https://crawlbrulee.com/docs/scrape/async) · [webhooks](https://crawlbrulee.com/docs/scrape/webhooks) · [webhook verification](https://crawlbrulee.com/docs/webhook-verification)
