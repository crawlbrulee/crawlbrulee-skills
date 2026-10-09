---
name: crawlbrulee-api
description: use when you need crawlbrulee's shared api contract — the base url, bearer auth, the full endpoint list, the page_status_code field, what gets billed, the response_meta usage object, caching, zero data retention, proxy tiers, location targeting, and the error model. read this whichever interface you call from. also covers calling the http api directly with curl or a generated client.
license: Apache-2.0
metadata:
  author: crawlbrulee
  homepage: https://crawlbrulee.com
allowed-tools: Bash(curl:*)
---

# 🍮 crawlbrulee http api

this skill carries the parts of crawlbrulee that don't change with the interface — auth, the endpoint list, the page's status, billing, the response format, caching, zero data retention, proxies, and errors. the sdks, the cli, and the mcp are all thin wrappers over exactly this, so read it whatever you're calling from.

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

## the page's own status: `page_status_code`

**our http status says whether we did our job. the target site's status is part of the data.**

if the site really served a page, you get a `200` from us with that page — whatever status the site gave it. a `404`, `410`, `401`, `451` or `503` page comes back with its content, and the site's status sits in `page_status_code` at the top level of the result:

```jsonc
{
  "url": "https://example.com/old-page",
  "page_status_code": 404,              // what the site answered, after redirects
  "markdown": "# Page not found …",
  "response_meta": { "usage": { … } }
}
```

- **check `page_status_code` before you trust the content.** the markdown of a `404` page is the site's "not found" page, not the article you wanted. treat `page_status_code >= 400` as an error page from the site and say so — don't hand that text on as if it were real content.
- it is the status of the final page, after redirects. when the page is rendered in a browser, it is the status of the page itself, not of its images or scripts.
- it is on every scrape result: sync `POST /api/scrape`, `GET /api/scrape/result/{job_id}`, and the `data` of a successful `scrape.complete` webhook. map has no page status — a map reads many files, not one page.
- older responses may not carry the field yet. when it is missing, the page was served normally.

a non-2xx from us means something else: the request was wrong, we could not reach the site or get the real page, or something broke on our side. see [errors](#errors).

## what gets billed

errors are never billed. for a page the site served, the page's status decides:

- **we bill 2xx and 4xx pages, except 403, 407, 408, 429 and 451.**
- **5xx pages are never billed.**

so a `404` or `410` page costs the same as any page, and a `503` page costs 0. an unbilled page still comes back as a `200` with its content. its `total_credit_cost`, `engine_credit_cost`, `screenshot_slicing_credit_cost` and `zero_data_retention_credit_cost` are all `0`, while `engine`, `proxy` and `proxy_multiplier` still show how it was fetched.

## the `response_meta` object

every successful scrape carries `response_meta.usage`, so you can see what the call cost. the async status body (once `done`) and a successful `scrape.complete` webhook carry the same fields:

```jsonc
"response_meta": {
  "usage": {
    "total_credit_cost": 1,               // what this call cost you
    "engine_credit_cost": 1,              // engine base: http 1, browser 3, screenshot 5, cache 0
    "proxy_multiplier": 1,                // 1 for basic, 5 for advanced
    "screenshot_slicing_credit_cost": 0,  // screenshot slicing add-on: 0 or 1
    "zero_data_retention_credit_cost": 0, // zero data retention add-on: 0 or 1
    "engine": "http",                     // http, browser, screenshot, or cache
    "proxy": "basic"                      // the resolved tier that delivered it
  }
}
```

- **`total_credit_cost`** is what you were charged for this call. read it instead of predicting it.
- the other parts explain the price. this always holds: `total_credit_cost = engine_credit_cost × proxy_multiplier + screenshot_slicing_credit_cost + zero_data_retention_credit_cost`.
- **`engine_credit_cost`** is the engine base: `1` for `http` (a plain fetch, no JavaScript ran), `3` for `browser` (the page was rendered, as with `require_js: true`), `5` for `screenshot` (a capture was made), `0` for `cache`. it is also `0` when the page is not billed.
- **`proxy_multiplier`** is `1` for `basic` and `5` for `advanced`. it is always there, even when the engine cost is `0`.
- **`screenshot_slicing_credit_cost`** is `1` when this request cut a screenshot into slices, otherwise `0`. it is a flat +1, added after the multiplier, however many slices were made — also when `engine` is `cache`.
- **`zero_data_retention_credit_cost`** is `1` when `zero_data_retention: true` added its credit to a billed, fresh result (see [zero data retention](#zero-data-retention)), otherwise `0`. it is always there; on an older response it is missing, which means `0`.
- **`engine`** is the delivered engine. `engine: "cache"` identifies a cache hit.
- **`proxy`** is the **resolved** tier. if you asked for `auto`, this tells you which tier ran. it is never `auto`.

map also carries `response_meta`, with `pagination`, `truncation`, and a usage block with the same fields minus slicing: `total_credit_cost`, `engine_credit_cost`, `proxy_multiplier`, `zero_data_retention_credit_cost`, `engine`, `proxy`. for map, `total_credit_cost = engine_credit_cost × proxy_multiplier + zero_data_retention_credit_cost`. map `engine` is exactly `http` for fresh discovery or `cache` for a cached result; its resolved `proxy` is `basic` or `advanced`, never `auto`. `pagination` and `truncation` live **inside** `response_meta`, not at the top level.

what a call costs is at <https://crawlbrulee.com/pricing>. treat that page as the source of truth — `response_meta.usage` tells you the rest after the fact.

`/api/usage` is different: its `total_credits`, `used_credits` and `available_credits` are amounts for your whole account, not for one call.

## zero data retention

add `zero_data_retention: true` at the top level of the body (not inside `cache`) on `POST /api/scrape`, `POST /api/scrape/async` and `POST /api/map`. keeps the result out of the shared cache; anything stored to deliver it is kept for 24 hours, then deleted. it adds 1 credit and must be enabled for your organization. see [zero data retention](https://crawlbrulee.com/docs/zero-data-retention).

```bash
curl -X POST https://api.crawlbrulee.com/api/scrape \
  -H "Authorization: Bearer $CRAWLBRULEE_API_KEY" -H "Content-Type: application/json" \
  -d '{"url":"https://example.com","extract":{"markdown":true},"zero_data_retention":true}'
```

when it is not enabled, the request fails with a `403` `zero_data_retention_not_enabled`, not billed. the added credit shows as `zero_data_retention_credit_cost` in `response_meta.usage`.

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
- **`extract` is NOT part of the key.** adding or dropping an output field (say `raw_html`) on an otherwise identical request still matches the same entry. `extract.elements` too: a cache hit reads your selectors from the cached page, and still costs 0.
- **`require_js: true` only matches entries that were rendered with JavaScript.**
- **`cleanup` is part of the key**, so `ads_and_popups` and `exclude_selectors` both stay cacheable. requests with the same removals share an entry; different ones don't.
- a screenshot with a non-zero `actions_before` wait or scroll disables caching.

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

**branch on `name`, not on the status code** — a `403` can be `antibot_blocked` or `zero_data_retention_not_enabled`, for example. the names are stable. a page the site served is never an error — even a `404` or `503` page is a `200` with the site's status in `page_status_code` (see above). errors are never billed.

| `name` | what to do |
| --- | --- |
| `invalid_url`, `url_too_long`, `unsupported_url_schema`, `url_credentials_not_supported`, `blocked_url` | the url was rejected before we fetched it — fix the input. `blocked_url` also comes back when the site redirected to an address we don't fetch — retrying won't help |
| `validation_error` | the request body failed validation — this includes an invalid css selector, or too many selectors, in `extract.elements` |
| `invalid_credentials` | missing, expired, or revoked api key — a genuine key problem, not a transient one (see `service_unavailable`) |
| `access_denied` | the token can't reach this resource |
| `zero_data_retention_not_enabled` | HTTP 403 — you sent `zero_data_retention: true` but it is not enabled for your organization. not billed; send the request without it |
| `not_found` | unknown async job id, or a job submitted more than 24 hours ago |
| `too_many_requests` | you're going too fast, or the target site rate-limited us — back off, honoring `details.retry_after_ms` when it is there, and space out requests to that site |
| `usage_allocation_error` | credit or concurrency cap — `details.reason` says which (`credit_limit`, `concurrency_limit`, `duplicate_reservation`, `internal_error`) |
| `antibot_blocked` | the target's bot protection blocked us — verify your use is permitted and don't retry automatically |
| `too_many_redirects` | the target redirected the request in a loop, or through more hops than we follow (HTTP 422) — the target's doing, not a bad request; retrying rarely helps |
| `page_too_large` | the page's html was too large to process (HTTP 422) — terminal, the same url fails the same way; scrape a smaller page instead |
| `target_unreachable` | we could not reach the site at all (HTTP 502) — for example it did not answer in time, or its tls certificate was bad. no page came back, so there is no `page_status_code`. not billed. retrying later may help; if it keeps failing, check the url is right and the site is up |
| `unsupported_content` | the page's content type can't be extracted (HTTP 415) |
| `scrape_error` | most often: you asked for an async result before the job finished — poll `status` first. it can also mean we could not read your request, for example the body was not valid json |
| `unsupported_screenshot_output` | the request asked only for a screenshot and the page's content type can't be screenshotted — request another format, or drop the screenshot |
| `request_timeout` | took too long — safe to retry |
| `job_failed` | an async job ended in `failed` |
| `internal_server_error` | something went wrong on our side — retry. it can also happen when the site could not be reached, so check the url is right and the site is up before you tell us |
| `service_unavailable` | we couldn't look up your token or reach a dependency just then — your key is fine, back off and retry |

`details` is only present on `too_many_requests` (`retry_after_ms`, `limited_by`) and `usage_allocation_error` (`reason` plus current-vs-max numbers). rate-limit headers ride on every response — `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, and `Retry-After` on a 429. read those rather than hardcoding a rate; the published limits are in [rate limits](https://crawlbrulee.com/docs/rate-limits).

for general retrieval failures, `require_js: true` renders JavaScript content and `proxy: "advanced"` uses the higher-success proxy tier. an `antibot_blocked` response means the target's bot protection blocked us; it isn't a retry signal — a retry might succeed by applying more sophisticated parameters, i.e. the `advanced` proxy or `require_js: true`, but it isn't a guarantee. a `too_many_redirects` response (422) means the target redirected in a loop; that isn't one either — both `scrape` and `map` can return it. a `page_too_large` response (422) means the page's html was too large to process — terminal, so don't retry it; only `scrape` returns it. a `target_unreachable` response (502) means we could not reach the site at all — retry once after a pause, and stop if it keeps happening; both `scrape` and `map` can return it, though for `map` it is rare.

a page the site answered with an error status is **not** in this list: it is a `200` with `page_status_code`. a `5xx` page is often temporary on the site's side, and it costs 0, so a retry later is cheap. a `404` or `410` page is the site's real answer — retrying won't change it.

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
