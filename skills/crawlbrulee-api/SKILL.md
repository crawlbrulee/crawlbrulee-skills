---
name: crawlbrulee-api
description: use when you need crawlbrulee's shared api contract — the base url, bearer auth, the full endpoint list, the response_meta usage object, caching, proxy tiers, location targeting, and the error model. read this whichever interface you call from. also covers calling the http api directly with curl or a generated client.
license: Apache-2.0
metadata:
  author: crawlbrulee
  homepage: https://crawlbrulee.com
allowed-tools:
  - Bash(curl *)
---

# 🍮 crawlbrulee http api

this skill carries the parts of crawlbrulee that don't change with the interface — auth, the endpoint list, the response format, caching, proxies, and errors. the sdks, the cli, and the mcp are all thin wrappers over exactly this, so read it whatever you're calling from.

it doubles as the raw-http guide: if there's no first-party client for your stack, or you want zero dependencies, everything here is callable with curl.

new to crawlbrulee? read the **crawlbrulee** skill first.

## base url & auth

base url: `https://api.crawlbrulee.com`

every request needs `Authorization: Bearer cwbl_…` (get a key at <https://dashboard.crawlbrulee.com>). request bodies are json — send `Content-Type: application/json`.

```bash
export CRAWLBRULEE_API_KEY="cwbl_…"
```

## endpoints

| method | path | purpose | skill |
| --- | --- | --- | --- |
| POST | `/api/scrape` | scrape a url, wait for the result | **crawlbrulee-scrape** |
| POST | `/api/scrape/async` | submit a background scrape → `202 { job_id }` | **crawlbrulee-scrape-async** |
| GET | `/api/scrape/status/{job_id}` | async job state | **crawlbrulee-scrape-async** |
| GET | `/api/scrape/result/{job_id}` | async job result (same shape as a sync scrape) | **crawlbrulee-scrape-async** |
| POST | `/api/map` | discover a site's urls | **crawlbrulee-map** |
| GET | `/api/usage` | billing-cycle snapshot | below |
| GET | `/api/whoami` | org + token identity | below |

## the `response_meta` object

every successful scrape carries `response_meta.usage` with the full billing shape below. async
status and successful `scrape.complete` webhooks use the same usage fields:

```jsonc
"response_meta": {
  "usage": {
    "credits": 1,             // engine base × proxy multiplier + slices
    "engine": "http",         // http, browser, screenshot, or cache
    "proxy": "basic",          // the resolved tier that delivered it
    "screenshot_slices": 0     // flat slice add-on: 0 or 1
  }
}
```

- `credits` is what you were actually charged for this call — read it instead of predicting it.
- `engine` is the delivered billing base: `http` (1 credit — a plain fetch, no JavaScript ran), `browser` (3 — the page was rendered, as with `require_js: true`), `screenshot` (5 — a capture was made), or `cache` (0). `engine: "cache"` identifies a cache hit.
- `proxy` is the **resolved** tier. if you asked for `auto`, this tells you which tier ran. it is never `auto`.
- `screenshot_slices` is `1` when this request produced screenshot slices, otherwise `0`. it adds one credit to the engine base × proxy multiplier, including when `engine` is `cache`.
- for scrape, the billing formula is `credits = engine base × proxy multiplier + screenshot_slices`; map has no slice add-on and uses `credits = engine base × proxy multiplier`. `basic` multiplies by 1 and `advanced` by 5. read the returned `credits` rather than recomputing it.

map also carries `response_meta`, with `pagination`, `truncation`, and a usage block containing
`credits`, `engine`, and `proxy`. map responses do not include `screenshot_slices`, because map
does not produce screenshots. map `engine` is exactly `http` for fresh discovery or `cache` for
a cached result; its resolved `proxy` is `basic` or `advanced`, never `auto`. `pagination` and
`truncation` live **inside** `response_meta`, not at the top level.

what a call costs, and how credits are counted, is at <https://crawlbrulee.com/pricing>. treat that page as the source of truth — `response_meta.usage` tells you the rest after the fact.

## proxy tiers

`proxy` accepts `basic`, `advanced`, or `auto`, and defaults to **`auto`**.

- **`basic`** — the standard pool. right for most pages.
- **`advanced`** — enhanced proxy tier with a higher success rate.
- **`auto`** (default) — starts on `basic` and escalates to `advanced` if that doesn't deliver. you're billed at whichever tier delivered.

`auto` reserves against the worst-case tier up front, so a low balance can reject an `auto` request that would ultimately have been cheap. pin `basic` when you know a page is easy and you're watching your balance.

cached results are proxy-agnostic — a result cached from a `basic` fetch will serve an `advanced` request and vice versa.

## location

| field | applies to | notes |
| --- | --- | --- |
| `location.locale` | scrape only | bcp-47, e.g. `en-US`. sets `Accept-Language` and `navigator.language`. |
| `location.country` | scrape and map | iso 3166-1 alpha-2, case-insensitive (`us`, `DE`), or the regional values `eu` / `europe`. picks the proxy exit country. |

`eu` and `europe` are **not** aliases: `eu` exits from an EU member state; `europe` exits from anywhere in Europe, including non-eu countries like the uk, Switzerland, and Norway. pick `eu` when the distinction matters to you.

## caching

caching is on by default, and a fully cached result is **free** — 0 credits. you're only charged for the parts still computed fresh, such as a newly produced screenshot-slice variant of a cached capture — see <https://crawlbrulee.com/pricing>. control it with `cache`:

| field | applies to | notes |
| --- | --- | --- |
| `cache.max_age` | scrape and map | seconds, or an iso-8601 datetime meaning "only accept results cached after this". `0` bypasses the cache and forces a fresh fetch. `max_age` is the only cache field — `cache` rejects anything else. |

the default freshness windows differ between scrape and map — see [caching](https://crawlbrulee.com/docs/scrape/caching) for the current values and the full key-normalization rules.

things worth knowing about the cache key:

- known tracking params are stripped before the page is fetched, so they never reach the target site; fragments and trailing slashes are normalized.
- **every other query param is part of the key.** `?lang=en` and `?lang=fr` are separate entries — strip params you don't need before you send the url.
- **screenshot settings are part of the key** — a different capture type, viewport, or device mode is a different entry.
- **`location.locale` is part of the key** (`location.country` is not) — a `de-DE` request never serves from an `en-US` entry.
- **`extract` is NOT part of the key.** adding or dropping an output field (say `raw_html`) on an otherwise identical request still matches the same entry.
- **`require_js: true` only matches entries that were rendered with JavaScript.**
- **`cleanup.exclude_selectors` disables caching** for that request. `cleanup.ads_and_popups` does not — it is part of the key instead, so both settings stay cacheable.
- a screenshot with a non-zero `actions_before` wait or scroll disables caching too.

## account

```bash
curl -s https://api.crawlbrulee.com/api/usage -H "Authorization: Bearer $CRAWLBRULEE_API_KEY"
# → { total_credits, used_credits, available_credits, used_quota_percent, max_concurrency, usage_reset }

curl -s https://api.crawlbrulee.com/api/whoami -H "Authorization: Bearer $CRAWLBRULEE_API_KEY"
# → { organization_name, token_name, token_preview }
```

`usage` is how you check remaining credits and your concurrency cap before a large job. `whoami` confirms which org and token a key belongs to — the token is only ever echoed as a masked preview.

## errors

non-2xx responses share one shape — a stable code in `name`, a human-readable `message`, and optional `details`:

```jsonc
{ "name": "too_many_requests", "message": "…", "details": { "retry_after_ms": 12000, "limited_by": "org" } }
```

**branch on `name`, not on the status code.** the codes are stable; statuses are not always what you'd guess (`scrape_error` mirrors the target site's status, so a 404 page gives you `scrape_error` at 404).

| `name` | what to do |
| --- | --- |
| `invalid_url`, `url_too_long`, `unsupported_url_schema`, `url_credentials_not_supported`, `blocked_url` | the url was rejected before we fetched it — fix the input |
| `validation_error` | the request body failed validation |
| `invalid_credentials` | missing, expired, or revoked api key — a genuine key problem, not a transient one (see `service_unavailable`) |
| `access_denied` | the token can't reach this resource |
| `not_found` | unknown async job id, or a result that has aged out |
| `too_many_requests` | you're going too fast — back off, honoring `details.retry_after_ms` |
| `usage_allocation_error` | credit or concurrency cap — `details.reason` says which (`credit_limit`, `concurrency_limit`, `duplicate_reservation`, `internal_error`) |
| `antibot_blocked` | the target's bot protection blocked us — verify your use is permitted and don't retry automatically |
| `too_many_redirects` | the target redirected the request in a loop, or through more hops than we follow (HTTP 422) — the target's doing, not a bad request; retrying rarely helps |
| `page_too_large` | the page's html was too large to process (HTTP 422) — terminal, the same url fails the same way; scrape a smaller page instead |
| `scrape_error`, `unsupported_content` | the fetch itself failed, or the content type can't be extracted |
| `unsupported_screenshot_output` | the request asked only for a screenshot and the page's content type can't be screenshotted — request another format, or drop the screenshot |
| `request_timeout` | took too long — safe to retry |
| `job_failed` | an async job ended in `failed` |
| `internal_server_error` | our side — retry, then tell us |
| `service_unavailable` | we couldn't look up your token or reach a dependency just then — your key is fine, back off and retry |

`details` is only present on `too_many_requests` (`retry_after_ms`, `limited_by`) and `usage_allocation_error` (`reason` plus current-vs-max numbers). rate-limit headers ride on every response — `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, and `Retry-After` on a 429. read those rather than hardcoding a rate; the published limits are in [rate limits](https://crawlbrulee.com/docs/rate-limits).

for general retrieval failures, `require_js: true` renders JavaScript content and `proxy: "advanced"` uses the higher-success proxy tier. an `antibot_blocked` response means the target's bot protection blocked us; it isn't a retry signal — a retry might succeed by applying more sophisticated parameters, i.e. the `advanced` proxy or `require_js: true`, but it isn't a guarantee. a `too_many_redirects` response (422) means the target redirected in a loop; that isn't one either — both `scrape` and `map` can return it. a `page_too_large` response (422) means the page's html was too large to process — terminal, so don't retry it; only `scrape` returns it.

## generate a client

the api is described by an OpenAPI 3.0 document. browse the interactive reference at <https://crawlbrulee.com/docs/api-reference.html>; the machine-readable spec is at <https://crawlbrulee.com/docs/openapi.json>.

```bash
npx openapi-typescript https://crawlbrulee.com/docs/openapi.json -o crawlbrulee.d.ts
```

generate from the spec rather than hand-rolling types — and prefer a first-party client where one exists (**crawlbrulee-sdk-js**, **crawlbrulee-sdk-python**, **crawlbrulee-cli**, **crawlbrulee-mcp**), since they already handle error mapping and async polling.

## see also

- capabilities: **crawlbrulee-scrape**, **crawlbrulee-screenshots**, **crawlbrulee-scrape-async**, **crawlbrulee-map**
- clients: **crawlbrulee-sdk-js**, **crawlbrulee-sdk-python**, **crawlbrulee-cli**, **crawlbrulee-mcp**
- docs: <https://crawlbrulee.com/docs> · pricing: <https://crawlbrulee.com/pricing>
