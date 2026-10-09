# changelog

all notable changes to the crawlbrulee agent skills and plugin are documented here.

this project follows [Semantic Versioning](https://semver.org). the version is the one in the plugin manifests.

## 1.3.0 (2026-10-09)

### added

- **`extract.elements`.** a new scrape option that picks named values out of a page by css selector and returns them as json in a top-level `elements` field — a title, a price, the next-page link, or one object per product card with `fields`. it costs no extra credits, also on a cache hit. the skills teach when to reach for it (you need specific values, not the whole page), with a short example. see [elements](https://crawlbrulee.com/docs/scrape/elements). covered in `crawlbrulee-scrape` (with the new `elements_truncated` warning and `elements` in `unsupported_fields`), `crawlbrulee-scrape-async`, `crawlbrulee-api`, and the overview in `crawlbrulee`. the readme shows a short example.
- **client names.** `extract.elements` and `elements` on the result in the js and python sdks (1.3.0), the same input on the `scrape` and `scrape_async` tools in the mcp (1.3.0), and the `--element <name=selector>` and `--elements <json>` flags in the cli (5.3.0).
- on a json, plain-text, xml or markdown page, `elements` comes back in `unsupported_fields`. a pdf or an image is a `415` `unsupported_content`.
- in the cli, `--element` / `--elements` on their own return only the elements (plus metadata unless `--no-metadata`). add `-m` or another content flag to get the page too.
- **the `screenshot_unavailable` warning.** a screenshot was asked for, but the page came back from the `http` engine without one. added to the warning table in `crawlbrulee-scrape` and to `crawlbrulee-screenshots`.
- `crawlbrulee-scrape` notes that the `metadata_truncated` warning is retired and no longer sent. it can still show up on results stored before it was retired.

### changed

- `cleanup.exclude_selectors` no longer turns the cache off. it is part of the cache key, so requests with the same removals share a cache entry. updated in `crawlbrulee-scrape` and `crawlbrulee-api`.
- the overview no longer says there is no structured extraction: you get any values you name by css selector. there is still no llm extraction.
- the plugin listings mention that you can get just the values you name by css selector.
- "what crawlbrulee does not do" in the overview now ends with a note: if you need one of those features, email contact@crawlbrulee.com, and we might build it.

## 1.2.1 (2026-10-07)

### changed

- **screenshot links expire.** `crawlbrulee-screenshots` says the image url is signed and expires 24 hours after the scrape (for an async scrape, 24 hours after submit), with signed links in the example. `crawlbrulee-scrape-async` says a job answers for 24 hours after submit, then `404`. the `not_found` rows in `crawlbrulee-api` and both sdk skills say the same.

## 1.2.0 (2026-10-05)

### added

- **zero data retention.** a new request option, `zero_data_retention`, on scrape, async scrape and map, with `zero_data_retention_credit_cost` in `response_meta.usage` (the total now includes it) and a `403` `zero_data_retention_not_enabled` error. it keeps the result out of the shared cache and must be enabled for your organization. see [zero data retention](https://crawlbrulee.com/docs/zero-data-retention). covered in `crawlbrulee-api`, `crawlbrulee-scrape`, `crawlbrulee-scrape-async`, `crawlbrulee-map`, `crawlbrulee-screenshots`, and the overview in `crawlbrulee`.
- **client names.** `zero_data_retention` and `ZeroDataRetentionNotEnabledError` in the js and python sdks (1.2.0), the `zero_data_retention` tool input in the mcp (1.2.0), and the `--zero-data-retention` flag in the cli (5.2.0).

### changed

- `blocked_url` (HTTP 400) also comes back when the site redirected to an address we don't fetch. retrying won't help. noted in the `crawlbrulee-api` error table.

### removed

- the deprecated `credits` and `screenshot_slices` in `response_meta.usage` are no longer mentioned, and the skills no longer explain how to fall back to them. read `total_credit_cost` and `screenshot_slicing_credit_cost`, which always had the same values.

## 1.1.0 (2026-09-30)

### added

- **`page_status_code`.** the skills teach that a page the site really served is a successful scrape, whatever its status. a 404, 410 or 503 page comes back with its content, and the site's status is in `page_status_code` at the top level of the result. agents are told to check it before they use the content. covered in `crawlbrulee-api`, `crawlbrulee-scrape`, `crawlbrulee-scrape-async` (the result and the `scrape.complete` webhook), `crawlbrulee-screenshots`, and each interface skill. `crawlbrulee-map` explains that a map has no page status, and that a site that answered only with statuses we don't bill (a `5xx`, for example), or not at all, gives an empty map that costs 0.
- **the `target_unreachable` error** (HTTP 502): we could not reach the site at all. not billed; retrying later may help. added to the error table in `crawlbrulee-api` and to the cli, mcp and sdk skills, with the new `TargetUnreachableError` class in both sdks.
- **what gets billed.** we bill 2xx and 4xx pages, except 403, 407, 408, 429 and 451. 5xx pages are never billed, and errors are never billed.
- **the price parts in `response_meta.usage`.** `total_credit_cost`, `engine_credit_cost`, `proxy_multiplier` and `screenshot_slicing_credit_cost`, where `total_credit_cost = engine_credit_cost × proxy_multiplier + screenshot_slicing_credit_cost`. map gets the same fields minus slicing. every usage example uses the new names.

### changed

- `scrape_error` is described as what it is: most often, the async result was asked for before the job finished.
- the cli skill shows the page status at the end of the text-mode usage line (cli 5.1.0) and that a 404 page exits `0`.
- the async skill says a job whose page was a 404 ends `done`, and that `failed` means no page came back.
- the skills say how to read older responses that don't carry the new fields yet.

### deprecated

- **`credits` and `screenshot_slices` in `response_meta.usage`.** use `total_credit_cost` and `screenshot_slicing_credit_cost`, which always have the same values. they will be removed in a future version.

## 1.0.0 (2026-09-21)

- first public release: ten skills and the plugin manifests, under Apache-2.0.
