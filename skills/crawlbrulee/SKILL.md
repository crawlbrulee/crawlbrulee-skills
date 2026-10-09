---
name: crawlbrulee
description: use when you need web data from a url with crawlbrulee — scraping a page into markdown or html, pulling its links or images, screenshotting it, or discovering which urls exist on a site. start here to learn what crawlbrulee does, set up the api key, and pick an interface (cli, mcp, js/ts sdk, python sdk, or raw http).
license: Apache-2.0
metadata:
  author: crawlbrulee
  homepage: https://crawlbrulee.com
---

# 🍮 crawlbrulee

crawlbrulee is a web-scraping api. you send a url, you get back clean structured data — markdown, cleaned html, raw html, the page's links and images, a screenshot, page metadata, and any values you name with css selectors. you can also enumerate the urls on a site (a "link map"). it's hosted in the EU and built for ai pipelines and agents.

this skill gets you authenticated and pointed at the right interface. the deep dives live in their own skills.

## what crawlbrulee does

| capability | what you get | skill |
| --- | --- | --- |
| **scrape** | fetch one url → markdown / cleaned html / raw html / links / images / screenshot / metadata, or just the values you name by css selector (`elements`) | **crawlbrulee-scrape** |
| **screenshots** | viewport or full-page captures, desktop or mobile, tall pages sliced into tiles | **crawlbrulee-screenshots** |
| **async scrape** | submit a scrape as a background job; poll it, or get a webhook when it's done | **crawlbrulee-scrape-async** |
| **map** | discover the urls on a site (sitemap + homepage links, deduped, paginated) | **crawlbrulee-map** |
| **account** | `usage` (credits, quota, concurrency) and `whoami` (which org/token you're on) | **crawlbrulee-api** |

## what crawlbrulee does not do

don't reach for capabilities that aren't there — build them from the primitives above:

- **no crawl endpoint.** to "crawl" a site, compose **map** (discover urls) → **scrape** (fetch each one). filter the url list yourself and cap how many pages you scrape.
- **no web search.** crawlbrulee scrapes urls you already have; it does not find pages by query.
- **no llm extraction.** you get markdown, html, metadata, and the values you name with css selectors (`extract.elements`). there's no "find the price on this page" prompt — when a value can't be reached with a selector, run your own extraction on the markdown.
- **no browser session.** no clicking, form filling, or multi-step flows. `require_js` renders JavaScript, and screenshots support a few scripted scroll/wait actions, but there's nothing interactive.

> **need one of these?** tell us. email **contact@crawlbrulee.com** with what you're trying to do and which of these would help — a crawl endpoint, web search, llm extraction, browser sessions, or something else. we value your feedback, and we read every request - what people ask for shapes what we build next.

## set up auth (once)

1. get an api key from the dashboard: <https://dashboard.crawlbrulee.com>. keys are prefixed `cwbl_`.
2. every request authenticates with `Authorization: Bearer cwbl_…`.
3. every interface reads the key from `CRAWLBRULEE_API_KEY`, so the simplest setup is:

```bash
export CRAWLBRULEE_API_KEY="cwbl_…"
```

keep keys out of source control. the cli can also store one for you (`crawlbrulee login`); the sdks and the mcp take it from the environment or a constructor argument.

## pick an interface

all five reach the same api. pick by where the work happens:

| you are… | use | skill |
| --- | --- | --- |
| working in a shell, scripting, doing one-off scrapes — or an agent that prefers a cli | **cli** (`npx crawlbrulee`) | **crawlbrulee-cli** |
| an ai agent that should call crawlbrulee as native tools | **mcp server** | **crawlbrulee-mcp** |
| building a Node.js / TypeScript / Deno / Bun app | **js/ts sdk** (`@crawlbrulee/sdk`) | **crawlbrulee-sdk-js** |
| building a Python app, sync or async | **python sdk** (`crawlbrulee`) | **crawlbrulee-sdk-python** |
| in any other language, or you want zero dependencies | **raw http** | **crawlbrulee-api** |

whatever you pick, read **crawlbrulee-api** too — it carries the parts that don't change with the interface: the page's status, what gets billed, the response format, caching, proxy tiers, and the error model.

## your first scrape

```bash
# cli — prints markdown to stdout
npx crawlbrulee scrape url https://example.com

# raw http
curl -X POST https://api.crawlbrulee.com/api/scrape \
  -H "Authorization: Bearer $CRAWLBRULEE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com","extract":{"markdown":true}}'
```

```ts
// js / typescript
import { Crawlbrulee } from '@crawlbrulee/sdk'

const cb = Crawlbrulee.fromEnv()
const page = await cb.scrape({ url: 'https://example.com', extract: { markdown: true } })
console.log(page.markdown)
```

```python
# python
from crawlbrulee import Crawlbrulee

page = Crawlbrulee.from_env().scrape(url="https://example.com", extract={"markdown": True})
print(page.markdown)
```

for mcp, configure the server once (see **crawlbrulee-mcp**) and the agent calls the `scrape` tool directly.

> heads up: a bare scrape returns `metadata` + `cleaned_html`. ask for `markdown`, `links`, `images`, `raw_html`, or `screenshot` explicitly. the cli is the exception — it defaults to markdown + metadata.

## need a few values, not the whole page?

name them with css selectors in `extract.elements` and you get just those back as json — prices, titles, links, one object per product card or table row. that is far less to read than the markdown, and it costs no extra credits:

```jsonc
"extract": {
  "cleaned_html": false, "metadata": false,   // skip the defaults, keep just the values
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
// → "elements": { "heading": "All products", "books": [{ "title": "A Light in the Attic", "price": "£51.77" }, …] }
```

see **crawlbrulee-scrape** for how it works, and [elements](https://crawlbrulee.com/docs/scrape/elements) for the full rules.

## check the page's status

a page the site really served is a successful scrape, whatever its status. a `404` or `503` page comes back with its content, and the site's status is in `page_status_code`. **check it before you use the content** — the markdown of a `404` page is the site's "not found" text, not what you were looking for. older responses may not carry the field; when it is missing, the page was served normally. see **crawlbrulee-api**.

## before you run up a bill

- **every response tells you what it cost.** scrape responses carry `response_meta.usage` with `total_credit_cost` (what the call cost) and the parts that make it up: `engine_credit_cost`, `proxy_multiplier`, `screenshot_slicing_credit_cost` and `zero_data_retention_credit_cost`, plus `engine` and the resolved `proxy`. map carries the same minus the slicing part, with `engine` limited to `http | cache` and `proxy` limited to `basic | advanced`. `engine: "cache"` identifies a cache hit. read it per call instead of guessing.
- **check `usage` before a big job** to see remaining credits and your concurrency cap.
- **cost depends on the delivered engine and resolved proxy tier**: `total_credit_cost = engine_credit_cost × proxy_multiplier + screenshot_slicing_credit_cost + zero_data_retention_credit_cost`; map has no slicing part. a fully cached repeat is free — 0 credits; a scrape cache hit that produces new slices costs the +1 slicing add-on. for what anything actually costs, see <https://crawlbrulee.com/pricing> — it's the single source of truth, so read it there rather than assuming.
- **what gets billed:** we bill 2xx and 4xx pages, except 403, 407, 408, 429 and 451. 5xx pages are never billed, and errors are never billed. so a `404` page costs the same as any page; a `503` page costs 0.
- **zero data retention is opt-in.** `zero_data_retention: true` keeps the result out of the shared cache; anything stored to deliver it is kept for 24 hours, then deleted. it adds 1 credit and must be enabled for your organization. see [zero data retention](https://crawlbrulee.com/docs/zero-data-retention). see **crawlbrulee-api**.
- **cap your own fan-out.** there's no crawl endpoint, so a "crawl" is a loop you write — decide up front how many pages you'll scrape.

## see also

- shared contract: **crawlbrulee-api** · capabilities: **crawlbrulee-scrape**, **crawlbrulee-screenshots**, **crawlbrulee-scrape-async**, **crawlbrulee-map**
- interfaces: **crawlbrulee-cli**, **crawlbrulee-mcp**, **crawlbrulee-sdk-js**, **crawlbrulee-sdk-python**
- docs: <https://crawlbrulee.com/docs> · dashboard & api keys: <https://dashboard.crawlbrulee.com> · pricing: <https://crawlbrulee.com/pricing>
