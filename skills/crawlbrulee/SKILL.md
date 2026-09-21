---
name: crawlbrulee
description: use when you need web data from a url with crawlbrulee — scraping a page into markdown or html, pulling its links or images, screenshotting it, or discovering which urls exist on a site. start here to learn what crawlbrulee does, set up the api key, and pick an interface (cli, mcp, js/ts sdk, python sdk, or raw http).
---

# 🍮 crawlbrulee

crawlbrulee is a web-scraping api. you send a url, you get back clean structured data — markdown, cleaned html, raw html, the page's links and images, a screenshot, and page metadata. you can also enumerate the urls on a site (a "link map"). it's hosted in the EU and built for ai pipelines and agents.

this skill gets you authenticated and pointed at the right interface. the deep dives live in their own skills.

## what crawlbrulee does

| capability | what you get | skill |
| --- | --- | --- |
| **scrape** | fetch one url → markdown / cleaned html / raw html / links / images / screenshot / metadata | **crawlbrulee-scrape** |
| **screenshots** | viewport or full-page captures, desktop or mobile, tall pages sliced into tiles | **crawlbrulee-screenshots** |
| **async scrape** | submit a scrape as a background job; poll it, or get a webhook when it's done | **crawlbrulee-scrape-async** |
| **map** | discover the urls on a site (sitemap + homepage links, deduped, paginated) | **crawlbrulee-map** |
| **account** | `usage` (credits, quota, concurrency) and `whoami` (which org/token you're on) | **crawlbrulee-api** |

## what crawlbrulee does not do

don't reach for capabilities that aren't there — build them from the primitives above:

- **no crawl endpoint.** to "crawl" a site, compose **map** (discover urls) → **scrape** (fetch each one). filter the url list yourself and cap how many pages you scrape.
- **no web search.** crawlbrulee scrapes urls you already have; it does not find pages by query.
- **no llm or structured-data extraction endpoint.** you get markdown, html, and metadata back; run your own extraction on top.
- **no browser session.** no clicking, form filling, or multi-step flows. `require_js` renders JavaScript, and screenshots support a few scripted scroll/wait actions, but there's nothing interactive.

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

whatever you pick, read **crawlbrulee-api** too — it carries the parts that don't change with the interface: the response format, caching, proxy tiers, and the error model.

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

## before you run up a bill

- **every response tells you what it cost.** scrape responses carry `response_meta.usage` = `{ credits, engine, proxy, screenshot_slices }`; map responses carry `{ credits, engine, proxy }`, with `engine` limited to `http | cache` and resolved `proxy` limited to `basic | advanced`. `engine: "cache"` identifies a cache hit. read it per call instead of guessing.
- **check `usage` before a big job** to see remaining credits and your concurrency cap.
- **cost depends on the delivered engine and resolved proxy tier**: for scrape, `credits = engine base × proxy multiplier + screenshot_slices`; map has no slice add-on. a fully cached repeat is free — 0 credits; a scrape cache hit that produces new slices costs the +1 slice add-on. for what anything actually costs, see <https://crawlbrulee.com/pricing> — it's the single source of truth, so read it there rather than assuming.
- **cap your own fan-out.** there's no crawl endpoint, so a "crawl" is a loop you write — decide up front how many pages you'll scrape.

## see also

- shared contract: **crawlbrulee-api** · capabilities: **crawlbrulee-scrape**, **crawlbrulee-screenshots**, **crawlbrulee-scrape-async**, **crawlbrulee-map**
- interfaces: **crawlbrulee-cli**, **crawlbrulee-mcp**, **crawlbrulee-sdk-js**, **crawlbrulee-sdk-python**
- docs: <https://crawlbrulee.com/docs> · dashboard & api keys: <https://dashboard.crawlbrulee.com> · pricing: <https://crawlbrulee.com/pricing>
