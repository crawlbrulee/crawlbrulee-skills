---
name: crawlbrulee-scrape
description: use when you need the crawlbrulee scrape contract — turning a url into markdown, cleaned html, raw html, links, images, or page metadata, or pulling named values (prices, titles, links, table rows) out of a page by css selector. covers every extract format and which are on by default, js rendering, removing parts of the page (nav, footers), cache and proxy options, and the full response shape including the page's own status (page_status_code), warnings and unsupported fields.
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
| `zero_data_retention` | boolean | `false` | keep the result out of the shared cache, +1 credit — see below |

`max_age` is the only cache field — `cache` rejects anything else.

**the url you send is the cache key.** known tracking params are stripped before the page is fetched, so they never reach the target site — see [caching](https://crawlbrulee.com/docs/scrape/caching#tracking-parameters). every **other** query param is part of the key: `?lang=en` and `?lang=fr` are separate entries. strip params that don't change the page before you send the url, or you pay full price for near-duplicates. `extract` is **not** part of the key — adding or dropping an output field (say `raw_html` or `elements`) still matches the same entry.

**`zero_data_retention: true`** goes at the top level of the body, not inside `cache`. keeps the result out of the shared cache; anything stored to deliver it is kept for 24 hours, then deleted. it adds 1 credit and must be enabled for your organization. see [zero data retention](https://crawlbrulee.com/docs/zero-data-retention).

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
| `elements` | — | named values picked out by css selector, as json — see [below](#pulling-specific-values-elements) |

naming any format does **not** switch the defaults off — `extract: { markdown: true }` gives you markdown *plus* metadata and cleaned html. set a default to `false` explicitly to drop it:

```jsonc
"extract": { "markdown": true, "cleaned_html": false, "metadata": false }   // markdown only
```

## pulling specific values: `elements`

when you need a few values from a page — a price, a title, the next-page link, every product in a list, a table's rows — ask for them by css selector instead of reading the whole page. `extract.elements` gives back clean json under names you pick. that is far less for you to read than the markdown, and it costs **no extra credits**, also on a cache hit.

each key is a name you choose. the value is either a selector string (you get the text of the first match) or an object:

| key | what it does |
| --- | --- |
| `selector` | required. a standard css selector, `:not()` and `:has()` included |
| `output` | `text` (default), `html` (the outer html) or `attribute` |
| `attribute` | the attribute to read when `output` is `attribute`. `href` and `src` come back as full urls |
| `all` | `true` returns every match as a list, not just the first |
| `fields` | use instead of `output`: name → selector (or object), read **inside each match**. one object per match; nests up to 3 levels |

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
        "price": ".price_color",
        "url": { "selector": "h3 a", "output": "attribute", "attribute": "href" }
      }
    },
    "next_page": { "selector": "li.next a", "output": "attribute", "attribute": "href" }
  }
}
```

the result has a top-level `elements` with the same names:

```jsonc
"elements": {
  "heading": "All products",
  "books": [
    { "title": "A Light in the Attic", "price": "£51.77", "url": "https://books.toscrape.com/catalogue/a-light-in-the-attic_1000/index.html" },
    { "title": "Tipping the Velvet", "price": "£53.74", "url": "https://books.toscrape.com/catalogue/tipping-the-velvet_999/index.html" }
    // … one object per book
  ],
  "next_page": "https://books.toscrape.com/catalogue/page-2.html"
}
```

- **use `fields` for a list of records.** fields look only inside each match, so a card's title, price and url stay together. for the same reason a field selector that starts with `+` or `~` matches nothing, and a field can't read the matched element itself: an attribute on the `<article>` you matched comes back `null`, so match one level higher instead. two separate `all: true` lists can differ in length (a match without the attribute is left out), so don't pair them up by position.
- **every name you asked for is always there**, at every level. no match is `null`, or `[]` with `all: true`.
- values are read **after `cleanup`**, from the same page as `links` and `images`. if they render client-side, add `require_js: true`.
- `<script>` and `<style>` can't be selected, so JSON-LD and inline script data are out of reach — ask for `raw_html` for those.
- tables are read the way a browser reads them, so `table > tbody > tr` works even when the page's html leaves out `<tbody>`.
- an invalid selector (a pseudo-element like `p::before` too) or too many selectors is a `400` `validation_error`, not billed.
- a list stops at 1,000 matches, and a request has limits on matches and size. when a value hits one, `warnings` has `elements_truncated`.
- not an html page (json, plain text, xml or markdown)? `elements` comes back in `unsupported_fields`. a pdf or an image is a `415` `unsupported_content` instead.

the full rules and limits are in the docs: [elements](https://crawlbrulee.com/docs/scrape/elements).

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
- both settings are part of the cache key, so both stay cacheable. requests with the same `exclude_selectors` share a cache entry; different ones don't.

## the response

fields you didn't request are simply absent.

```jsonc
{
  "requested_url": "http://example.com",  // your url, echoed verbatim
  "url": "https://example.com",           // the url actually scraped — canonical, after redirects
  "page_status_code": 200,                // the status the site answered with — check it
  "content_type": "text/html",
  "markdown": "…",
  "cleaned_html": "…",
  "raw_html": "…",
  "links": [{ "text": "Home", "href": "https://example.com/", "internal": true }],
  "images": [{ "url": "https://example.com/a.png?v=2", "alt": "…" }],
  "screenshot": { /* see crawlbrulee-screenshots */ },
  "metadata": { "title": "…", "description": "…", "og_title": "…", "favicon_url": "…" },
  "elements": { "heading": "All products", "books": [ /* … */ ] },  // only if you asked for elements
  "unsupported_fields": [],
  "warnings": [],
  "response_meta": {
    "usage": {
      "total_credit_cost": 1, "engine_credit_cost": 1, "proxy_multiplier": 1,
      "screenshot_slicing_credit_cost": 0, "zero_data_retention_credit_cost": 0,
      "engine": "http", "proxy": "basic"
    }
  }
}
```

- **check `page_status_code` before you trust the content.** a page the site really served comes back as a `200` from us, whatever its own status — so a `404`, `410`, `401`, `451` or `503` page arrives with its content, like any page. the markdown of a `404` page is the site's "not found" text. treat `page_status_code >= 400` as an error page from the site, and show that status rather than passing the text on as real content. it is the status of the final page, after redirects. older responses may not carry the field; when it is missing, the page was served normally.

- **you get both urls back.** `requested_url` is the url you sent, echoed verbatim — before any redirects. `url` is the url that was actually scraped: after redirects, in normalized form. links, images, and the `internal` flag are computed against `url`; use `requested_url` when you need to correlate a response with the url you submitted.
- **`links[].href` comes back exactly as it appears in the page** — absolute or relative, whatever the author wrote. resolve it yourself against `url` if you need an absolute link. `internal` tells you whether it points at the same site.
- **`images[].url` is always absolute** — we resolve document-relative `src`s against the page url and preserve query strings. links and images differ here deliberately; don't assume one behaves like the other.
- **`links` and `images` each have a per-page ceiling.** a page with an unusual number of either is cut at the cap rather than trimmed silently — you get the entries up to the ceiling plus a `links_truncated` or `inline_images_truncated` code in `warnings`. the current ceilings are in the [docs](https://crawlbrulee.com/docs/scrape).
- **`metadata`** fields are all optional and omitted when the page doesn't have them.
- **`response_meta`** is always present. read `usage.total_credit_cost` for what the call cost. a page whose status we don't bill (a `5xx`, `403`, `451`, …) shows `0` there, with every cost part `0`. see **crawlbrulee-api** for every usage field and the billing rule.

### `unsupported_fields`

if you request a format that doesn't apply to the content type — `metadata` of a json file, say, or `elements` on a json, plain-text, xml or markdown page — that field name comes back in `unsupported_fields` and the rest of your payload is delivered normally. it's not an error; check the array rather than assuming every requested field arrived.

### `warnings`

stable string codes for things worth flagging on an otherwise successful scrape. the values don't change, so you can switch on them:

| code | what it means |
| --- | --- |
| `screenshot_truncated` | the page was longer than the maximum capture height |
| `links_truncated` | the page had more links than we return for one page |
| `inline_images_truncated` | the page had more inline images than we return for one page |
| `raw_html_truncated` | the document was larger than the `raw_html` budget |
| `elements_truncated` | an `elements` value hit a limit: a list was cut at 1,000 matches, or the request reached its match or size limit (a value that didn't fit is `null`, never cut short) |

every one of them means you still got the output, cut at the ceiling — never a silent trim (an `elements` value too big to fit is the one exception: it comes back `null`). the current ceilings live in the [docs](https://crawlbrulee.com/docs/scrape).

a second family means that section's extraction failed, so the field came back omitted or empty while the rest of the scrape succeeded:

| code | what it means |
| --- | --- |
| `links_unavailable` | link extraction failed — `links` is omitted or empty |
| `inline_images_unavailable` | image extraction failed — `images` is omitted or empty |
| `metadata_unavailable` | metadata extraction failed — `metadata` is omitted or empty |
| `screenshot_unavailable` | a screenshot was asked for, but the page came back from the `http` engine without one — `screenshot` is omitted |

**an empty field carrying one of these does not mean the page had none.** that's the whole point of the code — it separates "we couldn't read them" from "there weren't any". don't report the absence as a finding; re-run the scrape. an empty field with no such warning is a real absence.

cache hits and async result fetches carry their warnings too. `metadata_truncated` is retired and no longer sent; it can still show up on results stored before it was retired.

## when a page comes back empty or a request fails

first look at `page_status_code`. if the site answered `404` or `410`, the page isn't there — that is the site's real answer, and no option will change it. if it answered with a `5xx`, the site had a problem; that page costs 0, and a retry later may get the real page.

if the status is fine but the content is thin, two levers, in this order:

1. **`require_js: true`** — use it when the content renders client-side.
2. **`proxy: "advanced"`** — use the enhanced proxy tier for a higher retrieval success rate.

an `antibot_blocked` error means the target's bot protection blocked the request; don't retry it automatically. a `too_many_redirects` error (422) means the target redirected the request in a loop; same rule. a `page_too_large` error (422) means the page's html was too large to process — terminal, so don't retry it; scrape a smaller page instead. a `target_unreachable` error (502) means we could not reach the site at all — no page came back and you are not billed; retry once after a pause, and if it keeps failing, check the url and whether the site is up. `require_js` adds latency; see <https://crawlbrulee.com/pricing> for how the proxy tier affects cost.

## see also

- shared contract: **crawlbrulee-api** · screenshots: **crawlbrulee-screenshots** · background jobs: **crawlbrulee-scrape-async**
- find urls to scrape: **crawlbrulee-map**
- drive it from: **crawlbrulee-cli**, **crawlbrulee-mcp**, **crawlbrulee-sdk-js**, **crawlbrulee-sdk-python**
- docs: <https://crawlbrulee.com/docs/scrape>
