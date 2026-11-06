---
name: crawlbrulee-mcp
description: use when an mcp-aware agent or editor should scrape pages (or pull named values out of them by css selector, or send their own headers and cookies with the request), map sites, run background scrape jobs, or check crawlbrulee credits as native tool calls. covers installing the `@crawlbrulee/mcp` stdio server and its seven tools.
license: Apache-2.0
metadata:
  author: crawlbrulee
  homepage: https://crawlbrulee.com
---

# 🍮 crawlbrulee mcp server

`@crawlbrulee/mcp` exposes crawlbrulee to mcp-aware agents as native tools, so the agent can scrape and map without any glue code. it's a thin stdio adapter over `@crawlbrulee/sdk`, `npx`-runnable with zero install.

this skill is the tool surface. for what the api actually does — extract formats, proxy tiers, caching, errors — read **crawlbrulee-api** and the capability skills.

## install

the server reads `CRAWLBRULEE_API_KEY` (a `cwbl_…` key) from its environment each time it starts. your user gets the key and puts it in their shell profile themselves, as in **crawlbrulee** (set up auth). never ask for the key in the chat, and never print it.

add the server with no key in its config, before or after the export:

```bash
claude mcp add crawlbrulee -- npx -y @crawlbrulee/mcp
```

hosts that read a json mcp config take the same thing as an entry: the command `npx` and the args `-y @crawlbrulee/mcp`.

where the variable doesn't reach the server on its own:

- **Codex** passes it only when it's listed. in `~/.codex/config.toml`, add `env_vars = ["CRAWLBRULEE_API_KEY"]` under `[mcp_servers.crawlbrulee]`.
- **Gemini CLI** passes it only when it's listed. in `~/.gemini/settings.json` (or `.gemini/settings.json` in a project), give the `crawlbrulee` entry under `mcpServers` the line `"env": { "CRAWLBRULEE_API_KEY": "$CRAWLBRULEE_API_KEY" }`.
- **Cursor and Windsurf:** name it to be sure. in `~/.cursor/mcp.json` (Cursor) or `mcp_config.json` (Windsurf, opened from its mcp settings), give the `crawlbrulee` entry under `mcpServers` the line `"env": { "CRAWLBRULEE_API_KEY": "${env:CRAWLBRULEE_API_KEY}" }`.
- **apps opened from the dock or the start menu**, like Claude Desktop, may not read the shell profile. there, your user writes the key into the app's config themselves. for Claude Desktop that's `~/Library/Application Support/Claude/claude_desktop_config.json` on macOS, `%APPDATA%\Claude\claude_desktop_config.json` on Windows.

the full entry for each host: <https://crawlbrulee.com/docs/mcp#install>.

other ways your user can enter the key, without the shell profile: the Claude Code plugin asks for it in a masked prompt; VS Code asks in a masked prompt with an `inputs` entry (`"password": true`, then `"CRAWLBRULEE_API_KEY": "${input:crawlbrulee-api-key}"` in the server's `env`); or they write it into the config themselves, in their own terminal (in Claude Code, not with the `!` prefix: that adds the command and its output to the chat).

when your user says it's done, call `whoami` to check the key works. the tools tell you what to do when the key is wrong: `missing_api_key` means your user needs to get a key, `invalid_credentials` means it's expired, revoked or mistyped and your user creates a new one. either way they put it in place themselves.

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
  "location": { "locale": "en-US", "country": "US" },
  "zero_data_retention": false // true keeps the result out of the shared cache, +1 credit — see below
}
```

`proxy` defaults to `auto` (starts basic, escalates to advanced on failure). screenshots come back as urls the agent fetches separately; in the rare case one can't be captured, the `screenshot` field is left out and the rest of the outputs still arrive — unless the screenshot was the *only* output requested, in which case the call errors (`unsupported_screenshot_output` for content types that can't be screenshotted) and isn't billed. scrape results carry `response_meta.usage`, so the agent can see per-call cost without calling `usage`: `total_credit_cost` is what the call cost, and `engine_credit_cost`, `proxy_multiplier`, `screenshot_slicing_credit_cost` and `zero_data_retention_credit_cost` are the parts that make it up. use `engine: "cache"` to identify a cache hit.

**`zero_data_retention: true`** (on `scrape`, `scrape_async` and `map`; needs `@crawlbrulee/mcp` `1.2.0` or newer) keeps the result out of the shared cache; anything stored to deliver it is kept for 24 hours, then deleted. it adds 1 credit and must be enabled for your organization. see [zero data retention](https://crawlbrulee.com/docs/zero-data-retention). only send it when the user asks for it.

**check `page_status_code` before you use the content.** a page the site really served is a normal `scrape` result, not a tool error — a `404`, `410` or `503` page comes back with its content, and the site's status is in `page_status_code` at the top level. the markdown of a `404` page is the site's "not found" text; tell the user the page answered `404` instead of treating that text as the page. `5xx` pages are never billed, and neither are `403`, `407`, `408`, `429` or `451` pages — see **crawlbrulee-api**. if a result has no `page_status_code`, it came from before the field existed, and the page was served normally.

**need a few values, not the whole page?** name them by css selector in `extract.elements` and the result has just those, as json, under a top-level `elements` (needs `@crawlbrulee/mcp` `1.3.0` or newer). far less for the agent to read than the markdown, and no extra credits:

```jsonc
{
  "url": "https://books.toscrape.com/",
  "extract": {
    "cleaned_html": false, "metadata": false,
    "elements": {
      "heading": "h1",
      "books": {
        "selector": "article.product_pod",
        "all": true,
        "fields": {
          "title": { "selector": "h3 a", "output": "attribute", "attribute": "title" },
          "price": ".price_color"
        }
      }
    }
  }
}
// → "elements": { "heading": "All products", "books": [{ "title": "A Light in the Attic", "price": "£51.77" }, …] }
```

`fields` are read inside each match, so each card's values stay together. a name with no match is `null` (`[]` for a list); `<script>` and `<style>` can't be selected; an invalid selector is a `validation_error`; `elements_truncated` in `warnings` means a value hit a limit; a json, plain-text, xml or markdown page lists `elements` in `unsupported_fields`. the same input works on `scrape_async`. see **crawlbrulee-scrape** and [targeted extraction](https://crawlbrulee.com/docs/scrape/targeted-extraction).

**your own headers and cookies.** `scrape` and `scrape_async` take `headers` (an object of name → value) and `cookies` (a list of `{ "name", "value" }`), sent to the site with the request (needs `@crawlbrulee/mcp` `1.4.0` or newer):

```jsonc
{
  "url": "https://app.example.com/orders",
  "extract": { "markdown": true },
  "headers": { "Authorization": "Bearer <short-lived token>" },
  "cookies": [{ "name": "session_id", "value": "<session>" }]
}
```

- **the values pass through the chat.** a tool call's input stays in the chat history, so ask your user for a short-lived token or session made for this, never a password or a long-lived key. say so before you ask. for long-lived secrets, use the cli or an sdk, where you can read them from an environment variable or a file.
- headers go only to the origin of `url` (or its https version, for an http url), cookies only to its host. the call skips the cache and is billed as a normal fresh scrape. a header you can't set is a `validation_error` whose message names it. your own `User-Agent` adds the `custom_user_agent` warning. `map` doesn't take them.
- a server older than `1.4.0` doesn't know these fields and drops them. if the `scrape` tool's input has no `headers` field, the server is too old: tell your user to update it. if the field is there and the result is still the site's login page, the token or session has likely expired.
- see **crawlbrulee-scrape** and [headers and cookies](https://crawlbrulee.com/docs/scrape/custom-headers-and-cookies).

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

`scrape_async` returns `{ "job_id": "…" }`. poll `scrape_status` until `done`, then call `scrape_result` — which returns the same shape as `scrape`, `page_status_code` included. `scrape_status` carries `response_meta.usage` once the job is `done`, and an `error` when it's `failed`. `scrape_result` errors if the job hasn't finished, so check status first. `done` means a page came back — a `404` page ends `done` too — so check `page_status_code` on the result. `failed` means no page came back.

with a `webhook` attached we deliver a single signed `scrape.complete` `POST` when the job finishes, with your `metadata` echoed back — react on completion instead of polling. see **crawlbrulee-scrape-async** for the lifecycle and signature verification.

### `map`

```jsonc
{
  "url": "https://example.com",
  "sitemap_only": false,
  "types": { "internal": true, "external": false, "internal_subdomains": true },
  "max_urls": 5000, // collection ceiling — default 5000, max 100000
  "page": 1,
  "limit": 1000, // page size — default 5000, max 10000
  "zero_data_retention": false // true keeps the result out of the shared cache, +1 credit
}
```

use it to plan which pages to scrape — there's no crawl tool, so a crawl is map plus repeated `scrape`. the response's `response_meta` carries `pagination`, `truncation`, and map usage (`total_credit_cost`, `engine_credit_cost`, `proxy_multiplier`, `zero_data_retention_credit_cost`, `engine`, `proxy`; nothing for screenshots). map has no `page_status_code`: a map that found urls is a normal result even if the home page was a `404`, and a site that answered only with statuses we don't bill (a `5xx`, for example), or not at all, gives an empty list that costs 0. map `engine` is `http` or `cache`, and its resolved `proxy` is `basic` or `advanced` (never `auto`).

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
| `missing_api_key` | `CRAWLBRULEE_API_KEY` isn't set in the mcp host's environment. your user needs to get a key and put it in place themselves |
| `crawlbrulee_error` | an api error without a typed name |
| `internal_error` | a bug in this mcp — please open an issue |

the ones worth handling: `zero_data_retention_not_enabled` (HTTP 403 — the organization doesn't have zero data retention; not billed, don't retry, send the request without the flag or point the user to sales@crawlbrulee.com), `too_many_requests` (back off), `usage_allocation_error` (out of credits or concurrency — show the user `usage`), `antibot_blocked` (the target's bot protection blocked the request — don't retry automatically), `too_many_redirects` (the target redirected in a loop — same, don't retry automatically), `page_too_large` (the page's html was too large to process — terminal, never retry it), `target_unreachable` (HTTP 502 — we could not reach the site at all, and it isn't billed; retry once after a pause, then check the url with the user), `request_timeout` (safe to retry), `service_unavailable` (a 503 from our side — back off and retry; the key is fine, don't prompt the user for a new one).

from mcp `1.1.0`, a `target_unreachable` error adds a short next step after the message; the `[target_unreachable]` prefix is the same, so branch on that. a `404` page is not in this list — it is a normal result with `page_status_code: 404`.

## when to prefer mcp

- the work happens **inside an agent session** and you want the model to decide when to scrape or map.
- you don't want to maintain client code — the tool schemas describe every parameter inline.

scripting in a shell instead? use **crawlbrulee-cli**. writing app code? use **crawlbrulee-sdk-js** or **crawlbrulee-sdk-python**.

## see also

- what the api does: **crawlbrulee-api**, **crawlbrulee-scrape**, **crawlbrulee-screenshots**, **crawlbrulee-scrape-async**, **crawlbrulee-map**
- the library underneath: **crawlbrulee-sdk-js**
- docs: <https://crawlbrulee.com/docs/mcp>
