---
name: crawlbrulee-screenshots
description: use when capturing a screenshot of a web page with crawlbrulee — viewport vs full-page, desktop vs mobile, custom viewport size and scale factor, ad and popup cleanup, scrolling or waiting before capture, slicing tall pages into tiles, and where the image url lands in the response.
---

# 🍮 crawlbrulee screenshots

you capture a screenshot as part of a scrape, through `extract.screenshot`. we don't inline the image — the response hands you a url you download separately, plus optional tiles for tall pages.

new to crawlbrulee? read the **crawlbrulee** skill first. for the surrounding request/response contract, read **crawlbrulee-scrape**.

## two things to know up front

**a screenshot always renders the page.** you don't need `require_js: true` to get one — asking for a capture implies rendering.

**in rare cases we can't capture one.** what happens then depends on what else you asked for. if the screenshot was one output among several, it's best-effort: the `screenshot` field is left out of the payload and everything else still arrives — check the field is there before reading `url` off it. if the screenshot was the **only** output you requested, the call errors instead of returning an empty success: `unsupported_screenshot_output` (a 422) when the content type can't be screenshotted — json, plain text, markdown, xml — or a 500 when the capture itself failed. neither error is billed. the same rule applies to async jobs when you fetch the result.

## options

`extract.screenshot` accepts:

| field | values | default | notes |
| --- | --- | --- | --- |
| `type` | `viewport` · `full_page` | **required** | `viewport` = above the fold; `full_page` = the whole scrollable page |
| `device_mode` | `desktop` · `mobile` | `desktop` | sets the user-agent and the default viewport |
| `viewport` | `{ width, height, device_scale_factor }` | per `device_mode` | see below |
| *(see `cleanup`, top level)* | object | **`{ ads_and_popups: true }`** | strips ads, cookie banners and popups before capture — a **top-level** field, not part of `screenshot` |
| `actions_before` | array, max 5 | `[]` | scroll/wait the page into shape first |
| `actions_after` | array, max 1 | `[]` | currently just `slice` |

**`viewport`** — if you pass the object at all, both `width` and `height` are required. they're integers in `[16, 10000]`. `device_scale_factor` is a number in `[1, 3]`, fractional allowed, default `1` — pass `2` for retina-density output. out-of-range values are rejected with a `400`.

**`cleanup` is a top-level request field, not a screenshot one.** it used to live inside `screenshot`; sending it there now is a 400. `cleanup.ads_and_popups` defaults to `true`, so banners are removed unless you ask for them — set it `false` only when you specifically want them in frame. the same block also shapes `markdown`, `cleaned_html`, `links` and `images`, and never `raw_html`.

## scrolling and waiting before capture (`actions_before`)

nudge a page into its final state before we shoot it — the fix for lazy-loaded images and on-scroll content. up to 5 actions, two types:

```jsonc
"actions_before": [
  { "type": "scroll", "pixels": 2000 },   // negative scrolls up
  { "type": "wait", "ms": 1500 }
]
```

there are caps on the total wait and total scroll distance across the array; exceeding either is a validation error. see [screenshots](https://crawlbrulee.com/docs/scrape/screenshots) for the current ceilings.

**a non-zero wait or scroll disables caching for that request** — every such call is a fresh fetch. zero-valued or empty arrays don't affect caching.

## slicing tall pages (`actions_after`)

a `full_page` capture of a long page can be enormous. add a single `slice` action to cut it into fixed-height tiles so each image stays manageable:

```jsonc
"actions_after": [{ "type": "slice", "height": 800 }]   // height ≥ 500
```

the response then carries a `slices[]` array alongside the main image. there's a cap on how many tiles we'll produce for one capture — see the docs. slices produced during this request add a flat 1 credit to the engine base × proxy multiplier, including when a cache hit needs a new slice variant. reusing an existing cached variant is free. check <https://crawlbrulee.com/pricing>, and read `response_meta.usage` — especially `credits` and `screenshot_slices` — to see what you were actually charged.

unlike `actions_before`, `actions_after` does **not** disable caching.

## example

```bash
curl -X POST https://api.crawlbrulee.com/api/scrape \
  -H "Authorization: Bearer $CRAWLBRULEE_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com",
    "extract": {
      "screenshot": {
        "type": "full_page",
        "device_mode": "desktop",
        "viewport": { "width": 1920, "height": 1080, "device_scale_factor": 2 },
        "actions_before": [{ "type": "scroll", "pixels": 4000 }],
        "actions_after": [{ "type": "slice", "height": 800 }]
      }
    }
  }'
```

## the response

```jsonc
{
  "screenshot": {
    "url": "https://cdn.crawlbrulee.com/screenshots/abc123.webp",
    "type": "full_page",
    "properties": {
      "file_name": "abc123.webp",
      "mime": "image/webp",
      "width": 1920,
      "height": 8400,
      "viewport": { "width": 1920, "height": 1080, "device_scale_factor": 2 }
    },
    "slices": [
      {
        "row_nr": 0,
        "url": "https://cdn.crawlbrulee.com/screenshots/abc123_slice_0.webp",
        "type": "slice",
        "properties": { /* file_name, mime, width, height, viewport */ }
      }
    ]
  }
}
```

- **the image is at a plain url on our cdn** — fetch it with an ordinary `GET`, no auth header needed. it stays retrievable for a while rather than forever; see the docs for the current window, and download promptly if you need to keep it.
- **`mime` is `image/webp`**, or `image/jpeg` for captures too tall for webp's format limit. it is never png — don't infer the format from the file extension, read `properties.mime`.
- **`slices[]` is only present when you asked for a `slice` action** — and asking doesn't guarantee it. in rare cases the capture is too large to slice, and we hand back the main image with no `slices` rather than failing the call. check the array is there before you read it. `row_nr` is zero-based and ordered top to bottom.
- **`properties.height`** is the real captured height, which for a long page can be very large.

## tips

- **check before you read.** when you requested other outputs alongside the screenshot, a failed capture just leaves the `screenshot` field absent — guard for it rather than assuming. a screenshot-only request that can't deliver errors instead, so there you handle the error, not a missing field.
- **very long pages get capped.** we stop at a maximum capture height and flag it with a `screenshot_truncated` entry in the response's `warnings` array. if you see that code, the image and its slices stop at the cap.
- **screenshot settings are part of the cache key.** the same url at a different capture type, viewport, or device mode is a different entry — changing any of them means a fresh fetch.
- **lazy content missing?** add an `actions_before` scroll and a short wait, or set `require_js: true` when the content renders client-side.

## see also

- the surrounding scrape: **crawlbrulee-scrape** · shared contract: **crawlbrulee-api**
- for long captures, submit in the background: **crawlbrulee-scrape-async**
- drive it from: **crawlbrulee-cli** (the `-ss` flag), **crawlbrulee-mcp**, **crawlbrulee-sdk-js**, **crawlbrulee-sdk-python**
- docs: <https://crawlbrulee.com/docs/scrape/screenshots>
