---
name: crawlbrulee-sdk-python
description: use when calling crawlbrulee from Python code, sync or async — the `crawlbrulee` PyPI package. covers the Crawlbrulee and AsyncCrawlbrulee clients, every method, background jobs with wait_for_scrape, webhook signature verification, and the typed error classes.
license: Apache-2.0
metadata:
  author: crawlbrulee
  homepage: https://crawlbrulee.com
---

# 🍮 crawlbrulee python sdk

the `crawlbrulee` package is the official Python client. fully typed (ships `py.typed`), sync **and** async clients, one runtime dependency (`httpx`), Python 3.10+.

this skill is the client surface. for what the api actually does — extract formats, proxy tiers, caching, errors — read **crawlbrulee-api** and the capability skills.

## install & construct

```bash
pip install crawlbrulee   # or: uv add crawlbrulee
```

```python
from crawlbrulee import Crawlbrulee, ScrapeExtract

client = Crawlbrulee(api_key="cwbl_…")
# or read CRAWLBRULEE_API_KEY from the environment:
client = Crawlbrulee.from_env()

page = client.scrape(url="https://example.com", extract=ScrapeExtract(markdown=True, links=True))
print(page.markdown)
print(page.metadata.title if page.metadata else None)
```

| option | default | notes |
| --- | --- | --- |
| `api_key` | — | sent as `Authorization: Bearer …`; required, or use `from_env()` |
| `base_url` | `https://api.crawlbrulee.com` | override the target host; trailing slashes stripped |
| `timeout` | `None` (**no timeout**) | per-request, in **seconds**; a per-call `timeout=` overrides it |

`from_env(**overrides)` reads the key from `CRAWLBRULEE_API_KEY` and forwards any other option through. it raises if the variable is unset or blank.

**the default is no timeout**, so a slow page can hang a request indefinitely. pass `timeout=` if you need a ceiling.

both clients are context managers and expose `close()` / `aclose()` to release the connection pool:

```python
with Crawlbrulee.from_env() as client:
    page = client.scrape(url="https://example.com")
```

## async client

`AsyncCrawlbrulee` mirrors the sync client method for method:

```python
import asyncio
from crawlbrulee import AsyncCrawlbrulee

async def main() -> None:
    async with AsyncCrawlbrulee.from_env() as client:
        page = await client.scrape(url="https://example.com", extract={"markdown": True})
        print(page.markdown)

asyncio.run(main())
```

## request inputs

top-level fields are keyword arguments. nested structures are typed dataclasses (importable from `crawlbrulee`) — or plain `dict`s, your choice:

```python
from crawlbrulee import ScrapeCleanup, ScrapeExtract, ScreenshotRequest

client.scrape(
    url="https://news.example.com/article-1",
    extract=ScrapeExtract(
        markdown=True,
        links=True,
        screenshot=ScreenshotRequest(type="full_page", device_mode="desktop"),
    ),
    require_js=True,
    proxy="advanced",
    cleanup=ScrapeCleanup(ads_and_popups=True, exclude_selectors=["nav", "footer"]),
    cache={"max_age": 3600},      # dataclass or dict
    location={"country": "US"},
)
```

`None`-valued options are dropped from the request entirely, so the server's defaults apply — passing `proxy=None` is the same as not passing `proxy`. a bare `scrape` returns metadata + cleaned html; **markdown is opt-in**.

## methods

each returns a typed dataclass and accepts a per-call `timeout=` in seconds.

| method | description |
| --- | --- |
| `scrape(url, **opts)` | scrape a url, block until done |
| `scrape_async(url, **opts)` | submit a background job → `{ job_id }` |
| `get_scrape_status(job_id)` | `pending` · `running` · `done` · `failed` |
| `get_scrape_result(job_id)` | result of a completed job (raises if not finished) |
| `wait_for_scrape(job_id, interval=2.0, timeout=300.0)` | poll until terminal, then return the result |
| `fetch_scrape_result_from_webhook(webhook)` | the result for a verified webhook body |
| `map(url, **opts)` | enumerate a site's urls |
| `usage()` | credits, quota, concurrency, reset |
| `whoami()` | org + token identity |

```python
job = client.scrape_async(url="https://example.com")
page = client.wait_for_scrape(job.job_id, interval=2.0, timeout=300.0)
```

`wait_for_scrape`'s `timeout` is the **overall wait budget**, not an http timeout (`timeout=0` waits forever). it raises `CrawlbruleeError` with `error_name="job_failed"` if the job fails, or `"request_timeout"` if the budget runs out.

pass `webhook=` to `scrape_async` to be called instead of polling — see **crawlbrulee-scrape-async**.

### reading the response

```python
page = client.scrape(url="https://example.com")

if page.response_meta:
    usage = page.response_meta.usage
    print(usage.credits, usage.engine, usage.proxy, usage.screenshot_slices)

if page.screenshot:
    print(page.screenshot.url)
```

two shapes to guard for: **`page.response_meta` is `Optional`** on a scrape (unlike `MapResponse.response_meta`, which is always there), and **`page.screenshot` is `Optional`** — when you requested other outputs too, a capture that couldn't be made leaves it `None` while the rest of the payload still arrives. `page.response_meta.usage` has `credits`, `engine`, `proxy`, and `screenshot_slices`; use `engine == "cache"` to identify a cache hit. a screenshot-only request that can't deliver raises instead (`error_name="unsupported_screenshot_output"` when the content type can't be screenshotted).

`map()` returns `MapUsage` under `response_meta.usage`: `credits`, `engine`, and `proxy`.
map `engine` is `http` or `cache`; its resolved `proxy` is `basic` or `advanced`, never
`auto`. map usage has no `screenshot_slices`.

## webhooks

```python
from crawlbrulee import verify_webhook_signature

result = verify_webhook_signature(
    payload=raw,               # the RAW bytes you received — not re-serialized json
    headers=request.headers,   # case-insensitive lookup
    secret=WEBHOOK_SECRET,
    tolerance_seconds=300,     # optional replay window; 0 disables the check
)

if not result.verified:
    return Response(status_code=400)   # result.reason says why
print(result.signed_with)              # "primary" | "rotated"
```

all parameters are keyword-only. it's pure crypto — no network, no extra dependency — and **returns a result instead of raising**, because a forged or replayed delivery is normal control flow. `reason` is one of `missing_signature`, `malformed_signature`, `timestamp_out_of_tolerance`, `signature_mismatch`.

then fetch the page — it accepts the decoded `dict` or a `ScrapeCompleteWebhook`:

```python
page = client.fetch_scrape_result_from_webhook(webhook)
```

it returns the result for a `success` job and raises for `failed` or `cancelled` ones. always verify **before** parsing or trusting the body. in FastAPI the raw body is `await request.body()`; in Flask it's `request.get_data()`. see **crawlbrulee-scrape-async**.

## errors

every failure subclasses `CrawlbruleeError`, which carries `status`, `error_name`, and `message`:

| class | when |
| --- | --- |
| `AuthenticationError` | 401 / 403 — missing, invalid, or unauthorized key |
| `AntibotBlockedError` | 403 `antibot_blocked` — the target site's bot protection blocked us; not a key problem, don't retry blindly |
| `TooManyRedirectsError` | 422 `too_many_redirects` — the target site redirected in a loop; not a bad request, don't retry blindly |
| `PageTooLargeError` | 422 `page_too_large` — the page's html was too large to process; terminal, don't retry it |
| `RateLimitError` | 429 — exposes `retry_after_ms`, `limited_by` |
| `UsageAllocationError` | credit or concurrency cap — exposes `reason`, `usage` |
| `ValidationError` | bad request (`invalid_url`, `url_too_long`, `blocked_url`, …) |
| `NotFoundError` | 404 (e.g. an unknown or aged-out job id) |
| `ServiceUnavailableError` | 503 — we're briefly unavailable; retryable, and never a reason to rotate the key |
| `TransportError` | network failure, timeout, non-json response |
| `CrawlbruleeError` | base class for anything else |

```python
import time
from crawlbrulee import Crawlbrulee, RateLimitError, UsageAllocationError

client = Crawlbrulee.from_env()
try:
    client.scrape(url="https://example.com")
except RateLimitError as err:
    time.sleep((err.retry_after_ms or 1000) / 1000)  # retry
except UsageAllocationError as err:
    print("plan limit hit:", err.reason, err.usage)
```

for exhaustive branching switch on `err.error_name`; `is_crawlbrulee_error(err)` is the type guard. the full error table is in **crawlbrulee-api**.

## see also

- what the api does: **crawlbrulee-api**, **crawlbrulee-scrape**, **crawlbrulee-screenshots**, **crawlbrulee-scrape-async**, **crawlbrulee-map**
- the js/ts equivalent: **crawlbrulee-sdk-js**
- docs: <https://crawlbrulee.com/docs/sdks/python>
