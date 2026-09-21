---
name: crawlbrulee-sdk-js
description: use when calling crawlbrulee from Node.js, TypeScript, Deno, or Bun code — the `@crawlbrulee/sdk` package. covers constructing the Crawlbrulee client, every method, background jobs with waitForScrape, webhook signature verification, the typed error classes, and cancellation with AbortSignal.
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
})

console.log(page.markdown)
console.log(page.metadata?.title)
console.log(page.response_meta.usage.credits, 'credits', page.response_meta.usage.engine, page.response_meta.usage.proxy)
```

`ScrapeRequest` and `ScrapeResponse` in the package's types carry inline docs for every field. two shapes worth knowing:

- **`page.response_meta` is required** — no guard needed to read `page.response_meta.usage.credits`.
- **`page.response_meta.usage` is `{ credits, engine, proxy, screenshot_slices }`**. `engine` is `http`, `browser`, `screenshot`, or `cache`; use `engine === 'cache'` to identify a cache hit. credits are the engine base × resolved proxy multiplier + the slice add-on.
- **`page.screenshot` is optional.** when you requested other outputs too, a capture that couldn't be made omits the field while the rest of the payload still arrives — guard with `page.screenshot?.url`. a screenshot-**only** request that can't deliver throws instead (`errorName: 'unsupported_screenshot_output'` when the content type can't be screenshotted).

`map()` returns `MapUsage` under `response_meta.usage`: `{ credits, engine, proxy }`. map
`engine` is `http` or `cache`; its resolved `proxy` is `basic` or `advanced`, never `auto`.
map usage has no `screenshot_slices`.

## background jobs

```ts
const { job_id } = await cb.scrapeAsync({ url: 'https://example.com' })

const page = await cb.waitForScrape(job_id, {
  intervalMs: 2000,          // poll every 2s (default)
  timeoutMs: 5 * 60 * 1000,  // give up after 5 min (default; 0 waits forever)
})
```

`waitForScrape`'s `timeoutMs` is the **overall wait budget**, not a per-request timeout — the http timeout stays whatever you gave the constructor. it rejects with `errorName: 'job_failed'` if the job fails, or `'request_timeout'` if the budget runs out.

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
| `RateLimitError` | 429 — exposes `retryAfterMs`, `limitedBy` |
| `UsageAllocationError` | credit or concurrency cap — exposes `reason`, `usage` |
| `ValidationError` | bad request (`invalid_url`, `url_too_long`, `blocked_url`, …) |
| `NotFoundError` | 404 (e.g. an unknown or aged-out job id) |
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
