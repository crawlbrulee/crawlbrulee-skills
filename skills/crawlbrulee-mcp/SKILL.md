---
name: crawlbrulee-mcp
description: use when an mcp-aware agent or editor should scrape pages, map sites, run background scrape jobs, or check crawlbrulee credits as native tool calls. covers installing the `@crawlbrulee/mcp` stdio server and its seven tools.
license: Apache-2.0
metadata:
  author: crawlbrulee
  homepage: https://crawlbrulee.com
---

# 🍮 crawlbrulee mcp server

`@crawlbrulee/mcp` exposes crawlbrulee to mcp-aware agents as native tools, so the agent can scrape and map without any glue code. it's a thin stdio adapter over `@crawlbrulee/sdk`, `npx`-runnable with zero install.

this skill is the tool surface. for what the api actually does — extract formats, proxy tiers, caching, errors — read **crawlbrulee-api** and the capability skills.

## install

set `CRAWLBRULEE_API_KEY` (a `cwbl_…` key from <https://dashboard.crawlbrulee.com>) in the server's environment.

most hosts take a stdio launch command. this is the whole command:

```bash
CRAWLBRULEE_API_KEY=cwbl_... npx -y @crawlbrulee/mcp
```

hosts that read a json mcp config take the same thing as an entry:

```jsonc
// add to the host's mcp config file
{
  "mcpServers": {
    "crawlbrulee": {
      "command": "npx",
      "args": ["-y", "@crawlbrulee/mcp"],
      "env": { "CRAWLBRULEE_API_KEY": "cwbl_..." }
    }
  }
}
```

if the host has its own "add mcp server" command, give it the same three parts: the command `npx`, the args `-y @crawlbrulee/mcp`, and the `CRAWLBRULEE_API_KEY` env var. per-host steps live in the [server's readme](https://github.com/crawlbrulee/crawlbrulee-mcp#readme).

the key is read on the first tool call rather than at startup, so a bad key surfaces as a clear tool error instead of a server that won't come up.

## tools

seven tools, all schema-described — the agent sees every parameter inline.

| tool | what it does |
| --- | --- |
| `scrape` | fetch one url, wait for the result |
| `scrape_async` | submit a background scrape → `{ job_id }` |
| `scrape_status` | look up a job: `pending` / `running` / `done` / `failed` |
| `scrape_result` | fetch a finished job's result |
| `map` | enumerate a site's urls |
| `usage` | credits, quota, concurrency, cycle reset — no arguments |
| `whoami` | org name, token name, masked token preview — no arguments |

### `scrape`

only `url` is required. a bare call returns metadata + cleaned html — **markdown is opt-in**.

```jsonc
{
  "url": "https://example.com",
  "extract": {
    "markdown": true,
    "links": true,
    "screenshot": { "type": "full_page", "device_mode": "desktop" }
  },
  "require_js": false,
  "proxy": "auto",
  "cleanup": { "ads_and_popups": true, "exclude_selectors": ["nav", "footer"] },
  "cache": { "max_age": 3600 },
  "location": { "locale": "en-US", "country": "US" }
}
```

`proxy` defaults to `auto` (starts basic, escalates to advanced on failure). screenshots come back as urls the agent fetches separately; in the rare case one can't be captured, the `screenshot` field is left out and the rest of the outputs still arrive — unless the screenshot was the *only* output requested, in which case the call errors (`unsupported_screenshot_output` for content types that can't be screenshotted) and isn't billed. scrape results carry `response_meta.usage` = `{ credits, engine, proxy, screenshot_slices }`, so the agent can see per-call cost without calling `usage`. use `engine: "cache"` to identify a cache hit.

see **crawlbrulee-scrape** and **crawlbrulee-screenshots** for what the fields mean.

### `scrape_async`, `scrape_status`, `scrape_result`

for slow pages or batches: submit, poll, fetch. `scrape_async` takes the same input as `scrape` plus an optional per-job completion `webhook`:

```jsonc
{
  "url": "https://example.com",
  "extract": { "markdown": true },
  "webhook": {
    "url": "https://hooks.example.com/cwbl",
    "metadata": { "ref": "order-42" }
  }
}
```

`scrape_async` returns `{ "job_id": "…" }`. poll `scrape_status` until `done`, then call `scrape_result` — which returns the same shape as `scrape`. `scrape_status` carries `response_meta.usage` once the job is `done`, and an `error` when it's `failed`. `scrape_result` errors if the job hasn't finished, so check status first.

with a `webhook` attached we deliver a single signed `scrape.complete` `POST` when the job finishes, with your `metadata` echoed back — react on completion instead of polling. see **crawlbrulee-scrape-async** for the lifecycle and signature verification.

### `map`

```jsonc
{
  "url": "https://example.com",
  "sitemap_only": false,
  "types": { "internal": true, "external": false, "internal_subdomains": true },
  "max_urls": 5000, // collection ceiling — default 5000, max 100000
  "page": 1,
  "limit": 1000 // page size — default 5000, max 10000
}
```

use it to plan which pages to scrape — there's no crawl tool, so a crawl is map plus repeated `scrape`. the response's `response_meta` carries `pagination`, `truncation`, and map usage (`credits`, `engine`, `proxy`; no `screenshot_slices`). map `engine` is `http` or `cache`, and its resolved `proxy` is `basic` or `advanced` (never `auto`).

**check `response_meta.truncation` before you treat the list as the whole site.** `limit` only trims this response — page for the rest. `max_urls` is different: discovery *stops* there, so a big site comes back as exactly 5000 links with `response_capped: false`, which looks complete and isn't. the honest signal is `discovery_capped: true` plus `discovery_cap_reason`:

- `"max_urls"` — **call `map` again with a higher `max_urls`**; this is the reason a retry is guaranteed to help.
- `"unread_files"` — some sitemap files couldn't be read this time (a `429`/`5xx`, an unreadable `.xml.gz`, a soft-404, a truncated file). often transient — **a retry may return more**, and a map with this reason is cached for only 15 minutes so a retry soon is worth it. if it keeps happening, the site is publishing something we can't read and the list won't grow.
- `"time"` · `"file_budget"` · `"depth"` · `"file_size"` — the site is too big, deep or slow for one pass; a bigger `max_urls` changes nothing. narrow the target or accept a partial list.
- `null` — nothing capped discovery; the list is complete.

`sitemaps_skipped` counts sitemap files skipped or read only in part. a retry with a bigger `max_urls` re-runs discovery and is billed as a fresh map — the short cached one is not reused.

returned links are `{ url }` and nothing else — there is no `source` field. each url comes back normalized, matching the url `scrape` reports. results are ordered by link type (internal → internal subdomains → external), then home-page links before sitemap-only ones, then shallower paths, then alphabetically — so page 1 carries a site's structure. see **crawlbrulee-map**.

## errors

a failed tool call returns an mcp error result (`isError: true`) with text in a stable format:

```
[<errorName>] <message> (HTTP <status>)
```

branch on the code. the set is the api's own error names (see **crawlbrulee-api** for the full table and what to do about each), plus three the mcp adds itself:

| code | meaning |
| --- | --- |
| `missing_api_key` | `CRAWLBRULEE_API_KEY` isn't set in the mcp host's environment |
| `crawlbrulee_error` | an api error without a typed name |
| `internal_error` | a bug in this mcp — please open an issue |

the ones worth handling: `too_many_requests` (back off), `usage_allocation_error` (out of credits or concurrency — show the user `usage`), `antibot_blocked` (the target's bot protection blocked the request — don't retry automatically), `too_many_redirects` (the target redirected in a loop — same, don't retry automatically), `page_too_large` (the page's html was too large to process — terminal, never retry it), `request_timeout` (safe to retry), `service_unavailable` (a 503 from our side — back off and retry; the key is fine, don't prompt the user for a new one).

## when to prefer mcp

- the work happens **inside an agent session** and you want the model to decide when to scrape or map.
- you don't want to maintain client code — the tool schemas describe every parameter inline.

scripting in a shell instead? use **crawlbrulee-cli**. writing app code? use **crawlbrulee-sdk-js** or **crawlbrulee-sdk-python**.

## see also

- what the api does: **crawlbrulee-api**, **crawlbrulee-scrape**, **crawlbrulee-screenshots**, **crawlbrulee-scrape-async**, **crawlbrulee-map**
- the library underneath: **crawlbrulee-sdk-js**
- docs: <https://crawlbrulee.com/docs/mcp>
