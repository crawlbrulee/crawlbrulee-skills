---
name: crawlbrulee-sdk-js
description: use when calling crawlbrulee from Node.js, TypeScript, Deno, or Bun code — the `@crawlbrulee/sdk` package. covers constructing the Crawlbrulee client, every method, background jobs with waitForScrape, webhook signature verification, the typed error classes, and cancellation with AbortSignal.
license: Apache-2.0
metadata:
  author: crawlbrulee
  homepage: https://crawlbrulee.com
---

# 🍮 crawlbrulee js/ts sdk

`@crawlbrulee/sdk` is the official TypeScript/JavaScript client. fully typed, ESM + CommonJS, zero runtime dependencies (just `fetch`). runs on Node.js 22+, Deno, Bun, and any `fetch`-capable runtime.

this skill is the client surface. for what the api actually does — extract formats, proxy tiers, caching, errors — read **crawlbrulee-api** and the capability skills.

## install & construct

```bash
npm install @crawlbrulee/sdk   # or pnpm add / yarn add
```

```ts
import { Crawlbrulee } from '@crawlbrulee/sdk'

const cb = new Crawlbrulee({ apiKey: 'cwbl_…' })
// or read CRAWLBRULEE_API_KEY from the environment:
const cb = Crawlbrulee.fromEnv()
```

| option | default | notes |
| --- | --- | --- |
| `apiKey` | — | sent as `Authorization: Bearer …`; required, or use `fromEnv()` |
| `baseUrl` | `https://api.crawlbrulee.com` | override the target host; trailing slashes stripped |
| `timeoutMs` | `0` (**no timeout**) | per-request timeout covering headers *and* body |

`fromEnv(overrides?)` reads the key from `CRAWLBRULEE_API_KEY` and forwards any other option through — `Crawlbrulee.fromEnv({ timeoutMs: 30_000 })`. it throws if the variable is unset or empty.

**the default is no timeout**, so a slow page can hang a request indefinitely. set `timeoutMs` if you need a ceiling.

## methods

every method takes an optional second argument `{ signal?, timeoutMs? }` for per-call overrides.

| method | returns |
| --- | --- |
| `scrape(request, options?)` | the scrape result |
| `scrapeAsync(request, options?)` | `{ job_id }` |
| `getScrapeStatus(jobId, options?)` | `pending` · `running` · `done` · `failed` |
| `getScrapeResult(jobId, options?)` | the result (throws if the job isn't finished) |
| `waitForScrape(jobId, options?)` | polls to a terminal state, returns the result |
| `fetchScrapeResultFromWebhook(webhook, options?)` | the result for a verified webhook body |
| `map(request, options?)` | the link map |
| `usage(options?)` | credits, quota, concurrency, reset |
| `whoami(options?)` | org + token identity |

```ts
const page = await cb.scrape({
  url: 'https://news.example.com/article-1',
  extract: {
    markdown: true,
    links: true,
    screenshot: { type: 'full_page', device_mode: 'desktop' },
  },
  require_js: true,
  proxy: 'advanced',
  cleanup: { ads_and_popups: true, exclude_selectors: ['nav', 'footer'] },
  cache: { max_age: 3600 },
  location: { locale: 'en-US', country: 'US' },
  zero_data_retention: false, // true keeps the result out of the shared cache — see below
})

if (page.page_status_code !== undefined && page.page_status_code >= 400) {
  console.warn(`the site answered ${page.page_status_code}`) // a 404 page is data, not an error
}
console.log(page.markdown)
console.log(page.metadata?.title)

const usage = page.response_meta.usage
console.log(usage.total_credit_cost, 'credits', usage.engine, usage.proxy)
```

`ScrapeRequest` and `ScrapeResponse` in the package's types carry inline docs for every field. shapes worth knowing:

- **a page the site served never throws, whatever its status.** a `404`, `410` or `503` page resolves like any page, with the site's status in `page.page_status_code`. check it before you use the content — the markdown of a `404` page is the site's "not found" text. the field is optional in the types because older responses don't carry it; when it is missing, the page was served normally.
- **`page.response_meta` is required** — no guard needed to read `page.response_meta.usage`.
- **`page.response_meta.usage`** has `total_credit_cost` (what the call cost) and its parts: `engine_credit_cost`, `proxy_multiplier`, `screenshot_slicing_credit_cost` and `zero_data_retention_credit_cost`, with `total_credit_cost = engine_credit_cost × proxy_multiplier + screenshot_slicing_credit_cost + zero_data_retention_credit_cost`. `engine` is `http`, `browser`, `screenshot`, or `cache`; use `engine === 'cache'` to identify a cache hit. the new fields are optional in the types because older responses don't send them.
- **`page.screenshot` is optional.** when you requested other outputs too, a capture that couldn't be made omits the field while the rest of the payload still arrives — guard with `page.screenshot?.url`. a screenshot-**only** request that can't deliver throws instead (`errorName: 'unsupported_screenshot_output'` when the content type can't be screenshotted).

`map()` returns `MapUsage` under `response_meta.usage`: `total_credit_cost`,
`engine_credit_cost`, `proxy_multiplier`, `zero_data_retention_credit_cost`, `engine`, `proxy`.
map `engine` is `http` or `cache`; its
resolved `proxy` is `basic` or `advanced`, never `auto`. map usage has nothing for screenshots,
and a map has no `page_status_code`.

`page_status_code`, the `*_credit_cost` fields, `proxy_multiplier` and `TargetUnreachableError`
are typed from `@crawlbrulee/sdk` `1.1.0`. an older release still returns the fields at runtime,
but its types don't name them — upgrade rather than casting.

**zero data retention.** `zero_data_retention: true` is accepted on `scrape()`, `scrapeAsync()` and `map()`; it keeps the result out of the shared cache (anything stored to deliver it is kept for 24 hours, then deleted) and adds 1 credit. it must be enabled for your organization, otherwise the call throws `ZeroDataRetentionNotEnabledError` (needs `@crawlbrulee/sdk` `1.2.0` or newer). see [zero data retention](https://crawlbrulee.com/docs/zero-data-retention).

## background jobs

```ts
const { job_id } = await cb.scrapeAsync({ url: 'https://example.com' })

const page = await cb.waitForScrape(job_id, {
  intervalMs: 2000,          // poll every 2s (default)
  timeoutMs: 5 * 60 * 1000,  // give up after 5 min (default; 0 waits forever)
})
```

`waitForScrape`'s `timeoutMs` is the **overall wait budget**, not a per-request timeout — the http timeout stays whatever you gave the constructor. it rejects with `errorName: 'job_failed'` if the job fails, or `'request_timeout'` if the budget runs out. a job whose page was a `404` doesn't fail — it resolves, and the result carries `page_status_code`.

pass a `webhook` to `scrapeAsync` to be called instead of polling. see **crawlbrulee-scrape-async**.

## webhooks

```ts
import { verifyWebhookSignature } from '@crawlbrulee/sdk'

const result = await verifyWebhookSignature({
  payload: rawBody,     // the RAW body (string or Uint8Array) — never re-serialized json
  headers: req.headers, // a fetch `Headers` or a plain object
  secret: process.env.CRAWLBRULEE_WEBHOOK_SECRET!,
  toleranceSeconds: 300, // optional replay window; 0 disables the check
})

if (result.verified) {
  console.log('signed with', result.signedWith) // 'primary' | 'rotated'
} else {
  console.warn('rejected:', result.reason)
}
```

it's **async** (built on Web Crypto, so it runs on Node.js 22+, browsers, Bun, Deno, and edge) and **returns a result rather than throwing** — a failed verification is normal control flow, not an exception. `reason` is one of `missing_signature`, `malformed_signature`, `timestamp_out_of_tolerance`, `signature_mismatch`.

then fetch the page:

```ts
const page = await cb.fetchScrapeResultFromWebhook(webhook)
```

it returns the result for a `success` job and throws for `failed` or `cancelled` ones. always verify **before** parsing or trusting the body. see **crawlbrulee-scrape-async** for the payload shape and rotation.

## errors

every failure extends `CrawlbruleeError`, which carries `status`, `errorName`, and `message`. typed subclasses cover the actionable cases:

| class | raised on |
| --- | --- |
| `AuthenticationError` | 401 / 403 — missing, invalid, or unauthorized key |
| `AntibotBlockedError` | 403 `antibot_blocked` — the target site's bot protection blocked us; not a key problem, don't retry blindly |
| `TooManyRedirectsError` | 422 `too_many_redirects` — the target site redirected in a loop; not a bad request, don't retry blindly |
| `PageTooLargeError` | 422 `page_too_large` — the page's html was too large to process; terminal, don't retry it |
| `TargetUnreachableError` | 502 `target_unreachable` — we could not reach the site at all; not billed, retrying later may help (from `1.1.0`; older releases raise a plain `CrawlbruleeError` with this `errorName`) |
| `ZeroDataRetentionNotEnabledError` | 403 `zero_data_retention_not_enabled` — `zero_data_retention` is not enabled for your organization; not billed (from `1.2.0`; older releases raise a plain `CrawlbruleeError` with this `errorName`) |
| `RateLimitError` | 429 — exposes `retryAfterMs`, `limitedBy` |
| `UsageAllocationError` | credit or concurrency cap — exposes `reason`, `usage` |
| `ValidationError` | bad request (`invalid_url`, `url_too_long`, `blocked_url`, …) |
| `NotFoundError` | 404 (e.g. an unknown job id, or one submitted more than 24 hours ago) |
| `ServiceUnavailableError` | 503 — we're briefly unavailable; retryable, and never a reason to rotate the key |
| `TransportError` | network failure, abort, timeout, non-json response |
| `CrawlbruleeError` | base class for anything else |

```ts
import { Crawlbrulee, RateLimitError, UsageAllocationError } from '@crawlbrulee/sdk'

try {
  await cb.scrape({ url: 'https://example.com' })
} catch (err) {
  if (err instanceof RateLimitError) {
    await sleep(err.retryAfterMs ?? 1000) // retry
  } else if (err instanceof UsageAllocationError) {
    console.error('plan limit hit:', err.reason, err.usage)
  } else {
    throw err
  }
}
```

for exhaustive branching switch on `err.errorName` — the literal union is exported as `ApiErrorName`, and `isCrawlbruleeError(err)` narrows an `unknown`. **`errorName` can be `null`** on a generic transport failure, so handle that case. the full error table is in **crawlbrulee-api**.

a `404` page is **not** an error, so no class above catches it — check `page.page_status_code` on the result instead.

## cancellation & timeouts

```ts
const controller = new AbortController()
const p = cb.scrape({ url: 'https://slow.example.com' }, { signal: controller.signal })
setTimeout(() => controller.abort(), 5_000)
```

the per-call `timeoutMs` and your `signal` compose — whichever fires first wins. precedence is per-call `timeoutMs` → constructor `timeoutMs` → the default (`0`, disabled). a fired timeout surfaces as `TransportError` with `errorName: 'request_timeout'`; an aborted signal as `'client_closed_request'`.

## see also

- what the api does: **crawlbrulee-api**, **crawlbrulee-scrape**, **crawlbrulee-screenshots**, **crawlbrulee-scrape-async**, **crawlbrulee-map**
- the Python equivalent: **crawlbrulee-sdk-python** · built on this sdk: **crawlbrulee-cli**, **crawlbrulee-mcp**
- docs: <https://crawlbrulee.com/docs/sdks/javascript-typescript>
