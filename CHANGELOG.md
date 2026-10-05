# changelog

all notable changes to the crawlbrulee agent skills and plugin are documented here.

this project follows [Semantic Versioning](https://semver.org). the version is the one in the plugin manifests.

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
