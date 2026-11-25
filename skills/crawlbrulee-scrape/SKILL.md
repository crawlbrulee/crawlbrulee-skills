---
name: crawlbrulee-scrape
description: use when you need the crawlbrulee scrape contract — turning a url into markdown, cleaned html, raw html, links, images, or page metadata, or pulling named values (prices, titles, links, table rows) out of a page by css selector, or saving the page as a pdf, or sending your own headers and cookies with the request, or clicking, scrolling, hovering and waiting on the page before it is read (actions). covers every extract format and which are on by default, js rendering, removing parts of the page (nav, footers), cache and proxy options, and the full response shape including the page's own status (page_status_code), warnings and unsupported fields.
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
| `extract` | object | `{ markdown, metadata }` | which formats you want — see below |
| `require_js` | boolean | `false` | go straight to a headless browser. without it, a page that needs one still gets one |
| `cleanup` | object | `{ ads_and_popups: true }` | what gets removed before any output is built — see below |
| `proxy` | `basic` · `advanced` · `auto` | `auto` | see **crawlbrulee-api** |
| `cache.max_age` | seconds or iso-8601 datetime | see docs | `0` forces a fresh fetch |
| `location.locale` | string | — | bcp-47, e.g. `en-US` |
| `location.country` | string | — | alpha-2, or `eu` / `europe` |
| `zero_data_retention` | boolean | `false` | keep the result out of the shared cache, +1 credit — see below |
| `headers` | object, name → string | — | your own http headers for the site, e.g. an api key or a fixed `User-Agent` the site has on its allowlist. see [below](#headers-and-cookies) |
| `cookies` | list of `{ name, value }` | — | your own cookies for the site. see [below](#headers-and-cookies) |
| `actions` | list of actions | — | click, scroll, hover and wait on the page before it is read. see [below](#actions-click-scroll-and-wait-first) |

`max_age` is the only cache field — `cache` rejects anything else.

**the url you send is the cache key.** known tracking params are stripped before the page is fetched, so they never reach the target site — see [caching](https://crawlbrulee.com/docs/scrape/caching#tracking-parameters). every **other** query param is part of the key: `?lang=en` and `?lang=fr` are separate entries. strip params that don't change the page before you send the url, or you pay full price for near-duplicates. `extract` is **not** part of the key — adding or dropping an output field (say `raw_html` or `elements`) still matches the same entry. the screenshot and pdf settings are the exception: a cached copy serves them only when it was made with the same settings (see **crawlbrulee-screenshots** and [the pdf section](#saving-the-page-as-a-pdf)). a request with `headers` or `cookies`, or with `actions`, skips the cache entirely (see [below](#headers-and-cookies)).

**`zero_data_retention: true`** goes at the top level of the body, not inside `cache`. keeps the result out of the shared cache; anything stored to deliver it is kept for 24 hours, then deleted. it adds 1 credit and must be enabled for your organization. see [zero data retention](https://crawlbrulee.com/docs/zero-data-retention).

for a background job instead of a blocking call, the body is identical — see **crawlbrulee-scrape-async**.

## extract formats

**this is the part people get wrong: `markdown` and `metadata` are on by default; everything else is opt-in.** omitting `extract` entirely gives you those two.

| field | default | what you get |
| --- | --- | --- |
| `metadata` | **`true`** | the parsed `<head>` block — title, description, og/twitter tags, favicon |
| `markdown` | **`true`** | clean markdown — the usual choice for llm/rag ingestion |
| `cleaned_html` | `false` | the page body with scripts, styles, ads and cookie banners removed |
| `raw_html` | `false` | the page's html with its classes and attributes, after `cleanup`, with its JSON-LD |
| `links` | `false` | the links on the page, up to a per-page cap |
| `images` | `false` | inline images, up to a per-page cap |
| `screenshot` | `false` | a capture — see **crawlbrulee-screenshots** |
| `pdf` | — | the page as a pdf document, as a signed link. see [below](#saving-the-page-as-a-pdf) |
| `elements` | — | named values picked out by css selector, as json — see [below](#pulling-specific-values-elements) |

naming any format does **not** switch the defaults off — `extract: { links: true }` gives you links *plus* markdown and metadata. set a default to `false` explicitly to drop it:

```jsonc
"extract": { "links": true, "markdown": false, "metadata": false }   // links only
```

the same goes for a screenshot, a pdf or `elements` on their own: send `markdown: false` and `metadata: false` with them.

## pulling specific values: `elements`

when you need a few values from a page — a price, a title, the next-page link, every product in a list, a table's rows — ask for them by css selector instead of reading the whole page. `extract.elements` gives back clean json under names you pick. that is far less for you to read than the markdown, and it costs **no extra credits**, also on a cache hit.

each key is a name you choose. the value is either a selector string (you get the text of the first match) or an object:

| key | what it does |
| --- | --- |
| `selector` | required. a standard css selector, `:not()` and `:has()` included |
| `output` | `text` (default), `html` (the outer html) or `attribute` |
| `attribute` | the attribute to read when `output` is `attribute`. `href` and `src` come back as full urls |
| `all` | `true` returns every match as a list, not just the first |
| `fields` | use instead of `output`: name → selector (or object), read **inside each match**; the selector `:scope` reads the match itself. one object per match; nests up to 3 levels |

```jsonc
"extract": {
  "markdown": false, "metadata": false,   // skip the defaults, keep just the values
  "elements": {
    "heading": "h1",
    "books": {
      "selector": "article.book",
      "all": true,
      "fields": {
        "title": { "selector": "h3 a", "output": "attribute", "attribute": "title" },
        "price": ".price",
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
  "heading": "Books",
  "books": [
    { "title": "The Night Garden", "price": "€18.90", "url": "https://shop.example.com/books/the-night-garden" },
    { "title": "Salt and Stone", "price": "€22.50", "url": "https://shop.example.com/books/salt-and-stone" }
    // … one object per book
  ],
  "next_page": "https://shop.example.com/books/?page=2"
}
```

- **use `fields` for a list of records.** fields look only inside each match, so a product's title, price and url stay together. for the same reason a field selector that starts with `+` or `~` matches nothing. to read the matched element itself, use the selector `:scope`: `{ "selector": ":scope", "output": "attribute", "attribute": "href" }` gives the matched element's own `href`. a field without `:scope` never answers with the match itself. two separate `all: true` lists can differ in length (a match without the attribute is left out), so don't pair them up by position.
- **every name you asked for is always there**, at every level. no match is `null`, or `[]` with `all: true`.
- values are read **after `cleanup`**, from the same page as `links` and `images`. if they render client-side, add `require_js: true`.
- JSON-LD blocks can be selected: `{ "selector": "script[type=\"application/ld+json\"]", "all": true }` gives each block's text as a string, for you to parse as json. other scripts and `<style>` can't be selected.
- tables are read the way a browser reads them, so `table > tbody > tr` works even when the page's html leaves out `<tbody>`.
- an invalid selector (a pseudo-element like `p::before` too) or too many selectors is a `400` `validation_error`, not billed.
- a list stops at 1,000 matches, and a request has limits on matches and size. when a value hits one, `warnings` has `elements_truncated`.
- not an html page (json, plain text, xml or markdown)? `elements` comes back in `unsupported_fields`. a pdf or an image is a `415` `unsupported_content` instead.

the full rules and limits are in the docs: [targeted extraction](https://crawlbrulee.com/docs/scrape/targeted-extraction).

## saving the page as a pdf

`extract.pdf` saves the page as a real pdf document: selectable text, working links, backgrounds printed, and no header or footer added. send `{}` for every default. it works on a sync scrape and on an async job.

```jsonc
"extract": {
  "markdown": false, "metadata": false,   // a pdf-only request
  "pdf": { "layout": "pages", "paper": "a4", "landscape": false, "media": "screen" }
}
```

| key | values | default | what it does |
| --- | --- | --- | --- |
| `layout` | `pages` · `single_page` | `pages` | `pages`: the page on sheets of paper, laid out as in the browser window and shrunk to fit the paper width. `single_page`: one page as wide as the browser window and as tall as the web page, up to 15,000 px; a taller page goes on over more pages, with the `pdf_single_page_overflow` warning |
| `paper` | `a4` · `letter` | `a4` | only with `layout: "pages"` |
| `landscape` | boolean | `false` | turn the paper sideways. only with `layout: "pages"` |
| `media` | `screen` · `print` | `screen` | `print` uses the site's own print style, which on many sites hides menus and images; a site without one looks as on screen |

the result has a top-level `pdf`:

```jsonc
"pdf": {
  "url": "https://cdn.crawlbrulee.com/…/page.pdf?token=…&expires=…",  // signed link
  "bytes": 184320                                                     // size of the file
}
```

- **download the file, don't keep the link.** the link is signed and expires 24 hours after the scrape (for an async job, 24 hours after you submitted it; fetching the result again doesn't extend it). use it exactly as returned.
- **it runs in a browser** and costs what a screenshot costs (the `screenshot` engine, 5 credits before the proxy multiplier). a pdf and a screenshot in one request are billed once. when the pdf can't be made, the request costs what it would cost without it.
- **with a screenshot in the same request,** both come from the same page in the same browser window: the pdf uses the screenshot's device and viewport, and its `color_scheme`, so a dark screenshot gives a dark pdf. in a request with a pdf and no screenshot, the pdf is made with the light theme.
- **no pdf came back?** in rare cases the pdf can't be made: it would be over 50 MB, it takes too long, or the page didn't load. the rest of the scrape comes back, `pdf` is left out, and `warnings` has `pdf_unavailable`. a pdf-only request fails instead: `422` `unsupported_pdf_output` when the content type (json, plain text, markdown, xml) can't be made into a pdf, or `500` when making it failed. neither is billed. a request for only a screenshot and a pdf gets `422` `unsupported_screenshot_output` instead. an async job that asks only for a pdf and gets none ends `failed`, not billed: its result answers the same `422`, or `400` `job_failed` when making the pdf failed.
- `paper` or `landscape` with `layout: "single_page"` is a `400` `validation_error`, not billed.
- **cache:** a cached copy answers only when it has a pdf made with the same settings (`layout`, `paper`, `landscape`, `media`, and with a screenshot the same device, viewport and `color_scheme`). `cache.max_age: 0` always makes a new one.

the full rules are in the docs: [pdf](https://crawlbrulee.com/docs/scrape/pdf).

## trimming the page

`cleanup` says what comes off the page before anything is built from it:

```jsonc
"cleanup": {
  "ads_and_popups": true,                                   // the default
  "exclude_selectors": ["nav", "footer", "aside", ".promo"]
}
```

`ads_and_popups` is **on by default** and removes ads, cookie banners, consent dialogs and chat widgets. turn it off with `"ads_and_popups": false` to get the page as-is.

`exclude_selectors` is for anything the default does not catch. at most 100 selectors, each at most 500 characters.

three things to know:

- it runs once, when the page has loaded, before any output is built, and before any `actions`. so it shapes every output: `markdown`, `cleaned_html`, `raw_html`, `links`, `images`, `metadata`, `elements`, the screenshot and the pdf. to get the page as it arrived, set `ads_and_popups` to `false` and send no `exclude_selectors`.
- something the page adds later, such as a popup that opens after a few seconds, is not removed.
- both settings are part of the cache key, so both stay cacheable. requests with the same `exclude_selectors` share a cache entry; different ones don't.

## headers and cookies

send your own `headers` (for example an api key, a bearer token, or a fixed `User-Agent` that the site has on its allowlist) and `cookies` with the scrape. they go to the site with the request.

```jsonc
{
  "url": "https://app.example.com/orders",
  "headers": { "Authorization": "Bearer <token>", "X-Tenant-Id": "acme" },
  "cookies": [{ "name": "session_id", "value": "<session>" }]
}
```

- `headers` is an object of header name → string value. `cookies` is a list of `{ "name", "value" }` objects, both strings, and the name can't be empty. both work on a sync scrape and on an async job, not on map.
- **where they go:** headers are sent only to the origin of `url` (same scheme, host and port), or to its https version when `url` is http, and not on a redirect to another origin. cookies go only to the host of `url`. cookies the site sets while redirecting are not kept.
- **never cached:** a request with headers or cookies never reads from or writes to the cache. it is billed as a normal fresh scrape, with no extra credits.
- **limits:** up to 10 headers and up to 20 cookies. each cookie is up to 4096 bytes (name and value), and headers and cookies together are up to 16384 bytes. values are printable ascii, with no space or tab at either end, and a cookie value may not hold `;`. a `__Secure-`, `__Host-` or `__Http-` cookie needs an https url.
- **headers you can't set:** headers a browser doesn't let a web page set can't be set here either, for security reasons. neither can headers that would make the site send back only part of the page, like `Range` or `If-None-Match`. send cookies in `cookies`, not a `Cookie` header, and the language in `location.locale`, not `Accept-Language`. a refused header is a `400` `validation_error` whose message names the header. the same header name twice (in any case) is refused too.
- **`User-Agent`:** you can set your own, but it can lower the success rate, and the response then carries the `custom_user_agent` warning. never put secrets in the `User-Agent`.
- a `Referer` must be the full http or https url of a public page, with no username or password.
- **use https:** on a plain `http://` url, headers and cookies travel unencrypted.
- header and cookie values stay out of our logs, and async status and result answers never show them.
- these values are your user's secrets. in code or the cli, read them from an environment variable or a file instead of writing them into the chat.

the full rules are in the docs: [headers and cookies](https://crawlbrulee.com/docs/scrape/custom-headers-and-cookies).

## actions: click, scroll and wait first

`actions` is a list of things to do on the page, in order, before we read it. every output shows the page as the actions left it.

```jsonc
{
  "url": "https://shop.example.com/books/",
  "actions": [
    { "type": "click", "selector": ".newsletter-popup .close", "optional": true },
    { "type": "click", "selector": "button.load-more" },
    { "type": "wait_for", "selector": "article.book:nth-of-type(40)" },
    { "type": "scroll", "to": "bottom" }
  ]
}
```

| type | fields | what it does |
| --- | --- | --- |
| `wait` | `ms` | waits that many milliseconds |
| `scroll` | `pixels`, or `to` (`top`, `bottom`, `left`, `right` or a css selector), optional `duration_ms` | scrolls the page; without `duration_ms` it moves at a steady pace, so images on the way load |
| `wait_for` | `selector` | waits up to 10 s for the element to be on the page and visible |
| `hover` | `selector`, or `x` and `y` | moves the mouse there; skipped on a mobile page |
| `click` | `selector` | clicks the first match (a tap on mobile) |

- **limits:** up to 12 actions and 60 s for the whole list. a list that breaks a rule (a bad selector, `pixels` and `to` together, waits over 60 s) is a `400` `validation_error`, not billed.
- **cleanup runs first.** ads, popups, cookie banners and your `exclude_selectors` are gone before the first action, so you can't click them. a popup that opens later can block a click: click its close button first, with `"optional": true`.
- **a failed action stops the list.** you still get the page as it was then, billed as usual, with the `actions_incomplete` warning. an `optional` action that fails is `skipped` and the list goes on.
- **the report:** the response has `actions`, one `{ type, status }` per action, in order. `status` is `ran`, `failed`, `skipped` or `not_run`.
- **a click that loads a new page** is followed: the rest of the list runs there, and the response is that page (its `url`, `page_status_code` and content). a click that opens a new tab is ignored.
- **when you know the new page's url** (a "next" link's `?page=2`, say), scrape that url directly instead of clicking to it. it loads once, not twice, so it's faster and more reliable; it can come from the cache, and it may not need a browser, so it can cost fewer credits.
- **runs in a browser, never cached.** billed as a browser scrape; the actions add no credits.
- **retries run the actions again**, from the start. use actions to read pages, never to place orders or sign up.
- **where you can send it:** `actions` works over the api directly and on the hosted mcp server. the sdks, the cli and the local mcp server don't have it yet; they keep the old name, `extract.screenshot.actions_before` (waits and scrolls), until their next update. the old name works until it is removed on 2027-02-01. never send both in one request (a `400`).

the full rules are in the docs: [actions](https://crawlbrulee.com/docs/scrape/actions).

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
  "pdf": { "url": "…", "bytes": 184320 },  // only if you asked for a pdf
  "metadata": { "title": "…", "description": "…", "og_title": "…", "favicon_url": "…" },
  "elements": { "heading": "Books", "books": [ /* … */ ] },  // only if you asked for elements
  "actions": [{ "type": "click", "status": "ran" }],        // only if you sent actions
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
| `pdf_single_page_overflow` | a `single_page` pdf was taller than the 15,000 px page limit, so it goes on over more pages |

every one of them means you still got the output, cut at the ceiling — never a silent trim (an `elements` value too big to fit is the one exception: it comes back `null`; and an overflowing pdf is whole, only on more pages). the current ceilings live in the [docs](https://crawlbrulee.com/docs/scrape).

a second family means that section's extraction failed, so the field came back omitted or empty while the rest of the scrape succeeded:

| code | what it means |
| --- | --- |
| `links_unavailable` | link extraction failed — `links` is omitted or empty |
| `inline_images_unavailable` | image extraction failed — `images` is omitted or empty |
| `metadata_unavailable` | metadata extraction failed — `metadata` is omitted or empty |
| `screenshot_unavailable` | a screenshot was asked for, but the page came back from the `http` engine without one — `screenshot` is omitted |
| `pdf_unavailable` | a pdf was asked for, but the page came back without one; `pdf` is omitted |

**an empty field carrying one of these does not mean the page had none.** that's the whole point of the code — it separates "we couldn't read them" from "there weren't any". don't report the absence as a finding; re-run the scrape. an empty field with no such warning is a real absence.

two more codes are about your request, not the page:

| code | what it means |
| --- | --- |
| `custom_user_agent` | the request set its own `User-Agent` header; this can lower the success rate. see [headers and cookies](#headers-and-cookies) |
| `actions_incomplete` | an action failed or never ran; the `actions` report says which, and the page is the one the actions left. see [actions](#actions-click-scroll-and-wait-first) |

new codes can be added over time, so treat a code you don't know as a note, not a failure.

cache hits and async result fetches carry their warnings too. `metadata_truncated` is retired and no longer sent; it can still show up on results stored before it was retired.

## when a page comes back empty or a request fails

first look at `page_status_code`. if the site answered `404` or `410`, the page isn't there — that is the site's real answer, and no option will change it. if it answered with a `5xx`, the site had a problem; that page costs 0, and a retry later may get the real page.

if the status is fine but the content is thin, two levers, in this order:

1. **`require_js: true`** — use it when parts of the page render client-side and the default scrape missed them.
2. **`proxy: "advanced"`** — use the enhanced proxy tier for a higher retrieval success rate.

if the content shows only after a click or a scroll (a "load more" button, a tab, lazy images), send [`actions`](#actions-click-scroll-and-wait-first).

an `antibot_blocked` error means the target's bot protection blocked the request; don't retry it automatically. a `too_many_redirects` error (422) means the target redirected the request in a loop; same rule. a `page_too_large` error (422) means the page's html was too large to process — terminal, so don't retry it; scrape a smaller page instead. a `target_unreachable` error (502) means we could not reach the site at all — no page came back and you are not billed; retry once after a pause, and if it keeps failing, check the url and whether the site is up. `require_js` adds latency; see <https://crawlbrulee.com/pricing> for how the proxy tier affects cost.

## see also

- shared contract: **crawlbrulee-api** · screenshots: **crawlbrulee-screenshots** · background jobs: **crawlbrulee-scrape-async**
- find urls to scrape: **crawlbrulee-map**
- drive it from: **crawlbrulee-cli**, **crawlbrulee-mcp**, **crawlbrulee-sdk-js**, **crawlbrulee-sdk-python**
- docs: <https://crawlbrulee.com/docs/scrape>
