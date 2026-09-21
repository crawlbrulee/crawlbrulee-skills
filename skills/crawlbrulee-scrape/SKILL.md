---
name: crawlbrulee-scrape
description: use when you need the crawlbrulee scrape contract — turning a url into markdown, cleaned html, raw html, links, images, or page metadata. covers every extract format and which are on by default, js rendering, excluding page furniture, cache and proxy options, and the full response shape including warnings and unsupported fields.
license: Apache-2.0
metadata:
  author: crawlbrulee
  homepage: https://crawlbrulee.com
---

# 🍮 crawlbrulee scrape

scrape fetches one url and returns the formats you asked for. this skill is the request/response contract, written in http/json terms — the shape is identical whichever interface you drive it from, so the sdks, cli, and mcp all map onto what's below.

new to crawlbrulee? read the **crawlbrulee** skill first. for the shared bits — auth, proxies, caching, errors — read **crawlbrulee-api**.

## the request

only `url` is required.

```bash
curl -X POST https://api.crawlbrulee.com/api/scrape \
  -H "Authorization: Bearer $CRAWLBRULEE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com",
    "extract": { "markdown": true, "links": true },
    "require_js": false,
    "proxy": "auto"
  }'
```

| field | type / values | default | notes |
| --- | --- | --- | --- |
| `url` | string | — | the url to scrape. http/https only. |
| `extract` | object | `{ metadata, cleaned_html }` | which formats you want — see below |
| `require_js` | boolean | `false` | render JavaScript in a headless browser |
| `cleanup` | object | `{ ads_and_popups: true }` | what gets removed before any output is built — see below |
| `proxy` | `basic` · `advanced` · `auto` | `auto` | see **crawlbrulee-api** |
| `cache.max_age` | seconds or iso-8601 datetime | see docs | `0` forces a fresh fetch |
| `location.locale` | string | — | bcp-47, e.g. `en-US` |
| `location.country` | string | — | alpha-2, or `eu` / `europe` |

`max_age` is the only cache field — `cache` rejects anything else.

**the url you send is the cache key.** known tracking params are stripped before the page is fetched, so they never reach the target site — see [caching](https://crawlbrulee.com/docs/scrape/caching#tracking-parameters). every **other** query param is part of the key: `?lang=en` and `?lang=fr` are separate entries. strip params that don't change the page before you send the url, or you pay full price for near-duplicates. `extract` is **not** part of the key — adding or dropping an output field (say `raw_html`) still matches the same entry.

for a background job instead of a blocking call, the body is identical — see **crawlbrulee-scrape-async**.

## extract formats

**this is the part people get wrong: `metadata` and `cleaned_html` are on by default; everything else is opt-in.** omitting `extract` entirely gives you those two.

| field | default | what you get |
| --- | --- | --- |
| `metadata` | **`true`** | the parsed `<head>` block — title, description, og/twitter tags, favicon |
| `cleaned_html` | **`true`** | main-content html, page furniture removed |
| `markdown` | `false` | clean markdown — the usual choice for llm/rag ingestion |
| `raw_html` | `false` | the unprocessed document |
| `links` | `false` | the links on the page, up to a per-page cap |
| `images` | `false` | inline images, up to a per-page cap |
| `screenshot` | `false` | a capture — see **crawlbrulee-screenshots** |

naming any format does **not** switch the defaults off — `extract: { markdown: true }` gives you markdown *plus* metadata and cleaned html. set a default to `false` explicitly to drop it:

```jsonc
"extract": { "markdown": true, "cleaned_html": false, "metadata": false }   // markdown only
```

## trimming the page

`cleanup` says what comes off the page before anything is built from it:

```jsonc
"cleanup": {
  "ads_and_popups": true,                                   // the default
  "exclude_selectors": ["nav", "footer", "aside", ".promo"]
}
```

`ads_and_popups` is **on by default** and removes ads, cookie banners, consent dialogs and chat widgets. Turn it off with `"ads_and_popups": false` when you want the page as-is — or when a site refuses to serve an ad-blocking client.

`exclude_selectors` is for anything the default does not catch. at most 100 selectors, each at most 500 characters.

three things to know:

- it shapes `markdown`, `cleaned_html`, `links`, `images` and the screenshot.
- it **never** touches `raw_html`. that is always the page as it arrived, before anything was removed — so you can always get back what we started from.
- `exclude_selectors` **disables caching** for that request, so every such call is a fresh fetch. `ads_and_popups` does not: both settings stay cacheable.

## the response

fields you didn't request are simply absent.

```jsonc
{
  "requested_url": "http://example.com",  // your url, echoed verbatim
  "url": "https://example.com",           // the url actually scraped — canonical, after redirects
  "content_type": "text/html",
  "markdown": "…",
  "cleaned_html": "…",
  "raw_html": "…",
  "links": [{ "text": "Home", "href": "https://example.com/", "internal": true }],
  "images": [{ "url": "https://example.com/a.png?v=2", "alt": "…" }],
  "screenshot": { /* see crawlbrulee-screenshots */ },
  "metadata": { "title": "…", "description": "…", "og_title": "…", "favicon_url": "…" },
  "unsupported_fields": [],
  "warnings": [],
  "response_meta": { "usage": { "credits": 1, "engine": "http", "proxy": "basic", "screenshot_slices": 0 } }
}
```

- **you get both urls back.** `requested_url` is the url you sent, echoed verbatim — before any redirects. `url` is the url that was actually scraped: after redirects, in normalized form. links, images, and the `internal` flag are computed against `url`; use `requested_url` when you need to correlate a response with the url you submitted.
- **`links[].href` comes back exactly as it appears in the page** — absolute or relative, whatever the author wrote. resolve it yourself against `url` if you need an absolute link. `internal` tells you whether it points at the same site.
- **`images[].url` is always absolute** — we resolve document-relative `src`s against the page url and preserve query strings. links and images differ here deliberately; don't assume one behaves like the other.
- **`links` and `images` each have a per-page ceiling.** a page with an unusual number of either is cut at the cap rather than trimmed silently — you get the entries up to the ceiling plus a `links_truncated` or `inline_images_truncated` code in `warnings`. the current ceilings are in the [docs](https://crawlbrulee.com/docs/scrape).
- **`metadata`** fields are all optional and omitted when the page doesn't have them.
- **`response_meta`** is always present — see **crawlbrulee-api**.

### `unsupported_fields`

if you request a format that doesn't apply to the content type — markdown of a pdf, say — that field name comes back in `unsupported_fields` and the rest of your payload is delivered normally. it's not an error; check the array rather than assuming every requested field arrived.

### `warnings`

stable string codes for things worth flagging on an otherwise successful scrape. the values don't change, so you can switch on them:

| code | what it means |
| --- | --- |
| `screenshot_truncated` | the page was longer than the maximum capture height |
| `links_truncated` | the page had more links than we return for one page |
| `inline_images_truncated` | the page had more inline images than we return for one page |
| `raw_html_truncated` | the document was larger than the `raw_html` budget |

every one of them means you still got the output, cut at the ceiling — never a silent trim. the current ceilings live in the [docs](https://crawlbrulee.com/docs/scrape).

a second family means that section's extraction failed, so the field came back omitted or empty while the rest of the scrape succeeded:

| code | what it means |
| --- | --- |
| `links_unavailable` | link extraction failed — `links` is omitted or empty |
| `inline_images_unavailable` | image extraction failed — `images` is omitted or empty |
| `metadata_unavailable` | metadata extraction failed — `metadata` is omitted or empty |

**an empty field carrying one of these does not mean the page had none.** that's the whole point of the code — it separates "we couldn't read them" from "there weren't any". don't report the absence as a finding; re-run the scrape. an empty field with no such warning is a real absence.

cache hits and async result fetches carry their warnings too.

## when a page comes back empty or a request fails

two levers, in this order:

1. **`require_js: true`** — use it when the content renders client-side.
2. **`proxy: "advanced"`** — use the enhanced proxy tier for a higher retrieval success rate.

an `antibot_blocked` error means the target's bot protection blocked the request; don't retry it automatically. a `too_many_redirects` error (422) means the target redirected the request in a loop; same rule. a `page_too_large` error (422) means the page's html was too large to process — terminal, so don't retry it; scrape a smaller page instead. `require_js` adds latency; see <https://crawlbrulee.com/pricing> for how the proxy tier affects cost.

## see also

- shared contract: **crawlbrulee-api** · screenshots: **crawlbrulee-screenshots** · background jobs: **crawlbrulee-scrape-async**
- find urls to scrape: **crawlbrulee-map**
- drive it from: **crawlbrulee-cli**, **crawlbrulee-mcp**, **crawlbrulee-sdk-js**, **crawlbrulee-sdk-python**
- docs: <https://crawlbrulee.com/docs/scrape>
