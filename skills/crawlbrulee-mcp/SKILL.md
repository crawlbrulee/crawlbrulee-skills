---
name: crawlbrulee-mcp
description: use when an mcp-aware agent or app (Claude, Claude Code, Cursor, Codex, Gemini CLI and others) should scrape pages, pull named values out of them by css selector, map sites, run background scrape jobs, or check crawlbrulee credits as native tool calls through the hosted server at https://api.crawlbrulee.com/mcp. covers connecting it with a browser sign-in (no api key), its seven tools, what they take and return, and its errors. the crawlbrulee plugin connects to this server.
license: Apache-2.0
metadata:
  author: crawlbrulee
  homepage: https://crawlbrulee.com
---

# 🍮 crawlbrulee mcp server

the hosted mcp server, `https://api.crawlbrulee.com/mcp`, gives mcp-aware agents crawlbrulee as native tools, so the agent can scrape and map without any glue code. nothing to install, and your user signs in through their browser instead of handling an api key. the crawlbrulee plugin connects to it.

need your own `headers` and `cookies` on a scrape, a `webhook` on a background job, `extract.raw_html`, or the screenshot's `viewport` and `actions`? the hosted server leaves those out. use the local server, `@crawlbrulee/mcp` (see **crawlbrulee-mcp-local**), the cli or an sdk.

this skill is the tool surface. for what the api actually does — extract formats, proxy tiers, caching, errors — read **crawlbrulee-api** and the capability skills.

## connect

the url is `https://api.crawlbrulee.com/mcp` (streamable http). the first time a host connects, it opens a crawlbrulee sign-in page in your user's browser. they sign in, pick the organization to connect and approve. you never see a key, and neither does the chat.

- **Claude Code:** `claude mcp add --transport http crawlbrulee https://api.crawlbrulee.com/mcp`, then your user runs `/mcp`, picks `crawlbrulee` and signs in. the crawlbrulee plugin adds the same server.
- **claude.ai and Claude Desktop:** your user adds a custom connector with the url and leaves the OAuth client id and secret empty.
- **Cursor:** `{ "mcpServers": { "crawlbrulee": { "url": "https://api.crawlbrulee.com/mcp" } } }` in `~/.cursor/mcp.json`.
- **Codex:** `url = "https://api.crawlbrulee.com/mcp"` under `[mcp_servers.crawlbrulee]` in `~/.codex/config.toml`, then `codex mcp login crawlbrulee`.
- **Gemini CLI:** `{ "mcpServers": { "crawlbrulee": { "httpUrl": "https://api.crawlbrulee.com/mcp" } } }` in `~/.gemini/settings.json`, then `/mcp auth crawlbrulee`.
- **other hosts** that take a remote server: add the url as a streamable http server; the host finds the sign-in on its own.

the full entry for each host: <https://crawlbrulee.com/docs/mcp/hosted>.

if your user has no account yet, they can sign up on the sign-in page. the free plan needs no credit card.

each connection gets its own key, which your user can see and delete with their other api tokens in the dashboard. deleting it disconnects the app, and the host asks your user to sign in again on its next call. a host that can't sign in can send an api key instead, as `Authorization: Bearer cwbl_…` in the server's headers; your user writes it into the config themselves. never ask for a key in the chat.

when your user says it's done, call `whoami` to check the connection.

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

a successful call returns the api's response as json text, the same body as the matching endpoint. a field a tool doesn't take is a `validation_error`, never dropped in silence.

### `scrape`

only `url` is required. a bare call returns markdown + metadata — **cleaned html is opt-in**.

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
  "location": { "locale": "en-US", "country": "US" },
  "zero_data_retention": false // true keeps the result out of the shared cache, +1 credit — see below
}
```

**`scrape` gives up after about 50 seconds** with a `request_timeout` error, before most chat apps cut a call off. for a slow page (heavy JavaScript, a full-page screenshot, a pdf), use `scrape_async` instead.

`proxy` defaults to `auto` (starts basic, escalates to advanced on failure). screenshots come back as urls the agent fetches separately; in the rare case one can't be captured, the `screenshot` field is left out and the rest of the outputs still arrive — unless the screenshot was the *only* output requested, in which case the call errors (`unsupported_screenshot_output` for content types that can't be screenshotted) and isn't billed. scrape results carry `response_meta.usage`, so the agent can see per-call cost without calling `usage`: `total_credit_cost` is what the call cost, and `engine_credit_cost`, `proxy_multiplier`, `screenshot_slicing_credit_cost` and `zero_data_retention_credit_cost` are the parts that make it up. use `engine: "cache"` to identify a cache hit.

**`zero_data_retention: true`** (on `scrape`, `scrape_async` and `map`) keeps the result out of the shared cache; anything stored to deliver it is kept for 24 hours, then deleted. it adds 1 credit and must be enabled for your organization. see [zero data retention](https://crawlbrulee.com/docs/zero-data-retention). only send it when the user asks for it.

**check `page_status_code` before you use the content.** a page the site really served is a normal `scrape` result, not a tool error — a `404`, `410` or `503` page comes back with its content, and the site's status is in `page_status_code` at the top level. the markdown of a `404` page is the site's "not found" text; tell the user the page answered `404` instead of treating that text as the page. `5xx` pages are never billed, and neither are `403`, `407`, `408`, `429` or `451` pages — see **crawlbrulee-api**.

**need a few values, not the whole page?** name them by css selector in `extract.elements` and the result has just those, as json, under a top-level `elements`. far less for the agent to read than the markdown, and no extra credits:

```jsonc
{
  "url": "https://shop.example.com/books/",
  "extract": {
    "markdown": false, "metadata": false,
    "elements": {
      "heading": "h1",
      "books": {
        "selector": "article.book",
        "all": true,
        "fields": {
          "title": { "selector": "h3 a", "output": "attribute", "attribute": "title" },
          "price": ".price"
        }
      }
    }
  }
}
// → "elements": { "heading": "Books", "books": [{ "title": "The Night Garden", "price": "€18.90" }, …] }
```

`fields` are read inside each match (`:scope` reads the match itself), so each product's values stay together. a name with no match is `null` (`[]` for a list); `<script>` and `<style>` can't be selected; an invalid selector is a `validation_error`; `elements_truncated` in `warnings` means a value hit a limit; a json, plain-text, xml or markdown page lists `elements` in `unsupported_fields`. the same input works on `scrape_async`. see **crawlbrulee-scrape** and [targeted extraction](https://crawlbrulee.com/docs/scrape/targeted-extraction).

**a pdf, and dark-mode screenshots.** `scrape` and `scrape_async` take `extract.pdf` (`{}` for every default) and `extract.screenshot.color_scheme` (`light` or `dark`), as in the api.

- the pdf comes back as a top-level `pdf` with a signed `url` and its size in `bytes`. the link expires 24 hours after the scrape: download the file if your user needs it later. it costs what a screenshot costs, and a pdf with a screenshot in one call is billed once.
- in the rare case the pdf can't be made, `pdf` is left out and `warnings` has `pdf_unavailable`. a pdf-only call errors instead (`unsupported_pdf_output` when the content type can't be made into a pdf) and isn't billed.
- `color_scheme: "dark"` asks the site for its dark theme, at no extra cost. a pdf in the same call shows it too; a pdf without a screenshot is made with the light theme.
- the options are in **crawlbrulee-scrape** (pdf) and **crawlbrulee-screenshots** (`color_scheme`).

see **crawlbrulee-scrape** and **crawlbrulee-screenshots** for what the fields mean.

### `scrape_async`, `scrape_status`, `scrape_result`

for slow pages or batches: submit, poll, fetch. `scrape_async` takes the same input as `scrape`.

`scrape_async` returns `{ "job_id": "…" }`. poll `scrape_status` with `{ "job_id": "…" }` until `done`, then call `scrape_result` — which returns the same shape as `scrape`, `page_status_code` included. `scrape_status` carries `response_meta.usage` once the job is `done`, and an `error` when it's `failed`. `scrape_result` errors if the job hasn't finished, so check status first. `done` means a page came back — a `404` page ends `done` too — so check `page_status_code` on the result. `failed` means no page came back, or a job that asked only for a screenshot or a pdf got none (not billed; `scrape_result` then errors with `job_failed`, or with `unsupported_screenshot_output` / `unsupported_pdf_output` when the content type can't be captured). status and result answer for 24 hours after the submit, then the job is gone (`not_found`). see **crawlbrulee-scrape-async** for the lifecycle.

### `map`

```jsonc
{
  "url": "https://example.com",
  "sitemap_url": "https://example.com/sitemap-products.xml", // optional: read this one sitemap file
  "depth": 3, // levels of nested sitemaps to read, 0 to 3, default 3
  "entries": "pages", // "pages" (default), "sitemaps" or "all"
  "modified_since": "2026-11-01T00:00:00Z", // optional: only entries changed since then
  "max_results": 5000, // collection ceiling, default 5000, max 100000
  "sitemap_only": false,
  "types": { "internal": true, "external": false, "internal_subdomains": true },
  "page": 1,
  "limit": 200, // page size — default 200 on this server, max 10000
  "zero_data_retention": false // true keeps the result out of the shared cache, +1 credit
}
```

**`map` returns 200 links a page** unless you set `limit`, so a big site doesn't flood the chat. page with `page` for the rest. a flagged or uncached map has no cached copy, so each later `page` runs and bills the map again: for a big list, set a higher `limit` once instead.

use it to plan which pages to scrape — there's no crawl tool, so a crawl is map plus repeated `scrape`. send only `url` for a plain map: it reads the site's top sitemap files and two nested levels, adds the home page links, and returns pages. the response's `response_meta` carries `pagination`, `truncation`, and map usage (`total_credit_cost`, `engine_credit_cost`, `proxy_multiplier`, `zero_data_retention_credit_cost`, `engine`, `proxy`; nothing for screenshots). map has no `page_status_code`: a map that found urls is a normal result even if the home page was a `404`, and a site that answered only with statuses we don't bill (a `5xx`, for example), or not at all, gives an empty list that costs 0. map `engine` is `http` or `cache`, and its resolved `proxy` is `basic` or `advanced` (never `auto`).

every link is `{ url, type, lastmod }`: `type` is `page`, or `sitemap` for a sitemap file you can read with `sitemap_url` in another call (a file on another site only when the site's robots.txt names its host); `lastmod` is the sitemap's date in utc, or `null` (always `null` for a home page link). there is no `source` field. each url comes back normalized, matching the url `scrape` reports. the most useful links come first, and the order is stable across pages.

**check `response_meta.truncation` before you treat the list as the whole site.** `limit` only trims this response — page for the rest. `max_results` is different: discovery *stops* there, so a big site comes back as exactly 5000 links with `response_capped: false`, which looks complete and isn't. the honest signal is `discovery_capped: true` plus `discovery_cap_reason`:

- `"max_results"` — **call `map` again with a higher `max_results`**; this is the reason a retry is guaranteed to help.
- `"depth"` — there are sitemap files below the `depth` you asked for. not a fault: they are in `unread_sitemaps`; read each with `sitemap_url`.
- `"unread_files"` — some sitemap files couldn't be read whole this time (a `429`/`5xx`, a file that is not a readable sitemap, a download that broke off: its links before the break come back). often transient: **a retry may return more**, so a retry soon is worth it. if it keeps happening, the site is publishing something we can't read and the list won't grow.
- `"time"` · `"file_budget"` · `"file_size"` — the site is too big or slow for one pass, or a file was too large; a bigger `max_results` changes nothing. read the files in `unread_sitemaps` with `sitemap_url` (no call reads a file that says it is over the size limit), or accept a partial list.
- `null` — nothing capped discovery; the list is complete.

`sitemaps_skipped` counts sitemap files skipped or read only in part, and `unread_sitemaps` lists the ones this call didn't read whole. a retry with a bigger `max_results` re-runs discovery and is billed as a fresh map: the smaller cached one is not reused.

**sitemaps one at a time.**

- **list a site's top sitemap files:** `{ "url": …, "depth": 0, "entries": "sitemaps" }`. they come back as links of `type: "sitemap"`, also in `unread_sitemaps`, with `discovery_cap_reason: "depth"`: expected, since `depth: 0` doesn't read them.
- **read one file:** `{ "url": …, "sitemap_url": <file> }`. no home page links are added.
- **a sitemap index returns no pages** at a low `depth` with the default `entries: "pages"`: it holds only sitemap files. an empty `links` with reason `"depth"` is the sign. ask with `entries: "sitemaps"` or a higher `depth`.
- **what changed since a date:** add `modified_since`. entries with no `lastmod` (and every home page link) are left out; a nested sitemap whose own `lastmod` is older isn't read, so a site with stale index dates can hide changes; and it never uses the cache, so every call and every `page` is billed as a fresh map.
- **a very large or deep site:** list the top files, read each with `sitemap_url` and `max_results: 100000`, and read every address in `unread_sitemaps` too, each once. when `unread_sitemaps` is full (1000), ask the same with `entries: "sitemaps"` to get every sitemap file as links.
- if the site answers `429` on the file in `sitemap_url`, its robots.txt or a top sitemap file, the call errors with `too_many_requests` (not billed): wait, then try again.

see **crawlbrulee-map** for every field and the full flow.

## errors

a failed call is a tool result with `isError: true`. its text is the api's error object as json:

```json
{ "name": "validation_error", "message": "extract.markdown: …" }
```

branch on `name`. it's the api's own error name (see **crawlbrulee-api** for the full table and what to do about each), and some errors add a `details` object. a `request_timeout` from `scrape` comes with a second text line that points to `scrape_async`. only an unknown tool name is an mcp protocol error.

the ones worth handling: `validation_error` (fix the call: the message names the field), `zero_data_retention_not_enabled` (HTTP 403 — the organization doesn't have zero data retention; not billed, don't retry, send the request without the flag or point the user to sales@crawlbrulee.com), `too_many_requests` (back off), `usage_allocation_error` (out of credits or concurrency — show the user `usage`), `antibot_blocked` (the target's bot protection blocked the request — don't retry automatically), `too_many_redirects` (the target redirected in a loop — same, don't retry automatically), `page_too_large` (the page's html was too large to process — terminal, never retry it), `target_unreachable` (HTTP 502 — we could not reach the site at all, and it isn't billed; retry once after a pause, then check the url with the user), `request_timeout` (use `scrape_async`, or retry), `service_unavailable` (a 503 from our side — back off and retry).

a `404` page is not in this list — it is a normal result with `page_status_code: 404`. a connection that was removed or expired doesn't reach the tools: the host asks your user to sign in again.

## when to prefer mcp

- the work happens **inside an agent session** and you want the model to decide when to scrape or map.
- you don't want to maintain client code — the tool schemas describe every parameter inline.

scripting in a shell instead? use **crawlbrulee-cli**. writing app code? use **crawlbrulee-sdk-js** or **crawlbrulee-sdk-python**.

## see also

- the local server, with an api key and a few more fields: **crawlbrulee-mcp-local**
- what the api does: **crawlbrulee-api**, **crawlbrulee-scrape**, **crawlbrulee-screenshots**, **crawlbrulee-scrape-async**, **crawlbrulee-map**
- docs: <https://crawlbrulee.com/docs/mcp>
