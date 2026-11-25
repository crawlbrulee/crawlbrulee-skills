# changelog

all notable changes to the crawlbrulee agent skills and plugin are documented here.

this project follows [Semantic Versioning](https://semver.org). the version is the one in the plugin manifests.

## unreleased

- **actions.** `crawlbrulee-scrape` teaches the top-level `actions` list: `wait`, `scroll`, `wait_for`, `hover`, `click`, `type` and `select`, run in order before the page is read. it shows how to fill in and send a form, such as a search box, and covers the limits (12 actions, 60 s), the `actions` report (`ok`, `failed`, `skipped`, `not_run`) and the `actions_incomplete` warning. `crawlbrulee-api` adds the lower rate limit for a request that types, and says a request with `actions` never uses the cache. `crawlbrulee-screenshots` moves scrolling and waiting before a capture to `actions`; `extract.screenshot.actions_before` is its old name, removed on 2027-02-01. `crawlbrulee-mcp` and `crawlbrulee-mcp-local` show the actions that read a page on `scrape` and `scrape_async` (on the local server from `@crawlbrulee/mcp` 2.0.0, whose screenshot no longer takes `actions_before`); filling in a form needs the http api. the overview lists actions.
- **`raw_html` comes after cleanup** and keeps the page's JSON-LD. `crawlbrulee-scrape` says cleanup shapes every output, and that `extract.elements` can read JSON-LD blocks; both mcp skills say so too.

## 1.6.5 (2026-11-23)

- crawlbrulee-mcp: the hosted server takes no `proxy` tier; every hosted call uses `auto`. the
  example input no longer sends one, and the list of what the hosted server leaves out names it.

## 1.6.4 (2026-11-23)

### changed

- no changes to the skills or the manifests' content. this version only carries the plugin to OpenAI's plugin directory again, which needs a new version number for each upload.

## 1.6.3 (2026-11-23)

### changed

- **codex plugin manifest:** the empty `screenshots` list is gone from `.codex-plugin/plugin.json`. the plugin had no screenshots to show, and OpenAI's plugin directory refuses the field.

## 1.6.1 (2026-11-23)

### changed

- **crawlbrulee-cli:** the pdf tip no longer shows a command that downloads the file. it says to download the pdf right away from the link the cli prints, since the link expires 24 hours after the scrape.

## 1.6.0 (2026-11-20)

### changed

- **map** (crawlbrulee-map, crawlbrulee-api, crawlbrulee-cli, crawlbrulee-mcp, crawlbrulee-mcp-local, the sdk skills): a sitemap file the map found to be a sitemap index comes back as type `sitemap_index`. with `modified_since`, every sitemap file is read whatever its own date, and each entry is judged by its own `lastmod`. a large site is mapped in several calls, starting with `depth: 1` and `entries: "sitemaps"`.
- **a page with status `444` is not billed** (crawlbrulee-api, crawlbrulee-scrape, crawlbrulee-mcp, crawlbrulee-mcp-local).
- **`proxy: "auto"` and `require_js`** (crawlbrulee-api, crawlbrulee-scrape, crawlbrulee-mcp, crawlbrulee-mcp-local): `auto`, the default, starts with the lightest way to get the page and steps up on its own when a site needs more: a stronger proxy, a real browser, or both. `require_js: true` goes straight to a browser; without it, a page that needs one still gets one.

## 1.5.0 (2026-11-18)

### added

- **pdf output: `extract.pdf`.** a scrape can save the page as a pdf: a real document with selectable text and working links. `{}` takes every default; `layout` (`pages` or `single_page`), `paper` (`a4` or `letter`), `landscape` and `media` (`screen` or `print`) set the rest. the result has a top-level `pdf` with a signed `url` that expires 24 hours after the scrape, and its size in `bytes`. it costs what a screenshot costs, and a pdf and a screenshot in one request are billed once. `crawlbrulee-scrape` has the options, the warnings `pdf_unavailable` and `pdf_single_page_overflow`, the new `422` `unsupported_pdf_output` for a pdf-only request whose content type can't be made into a pdf (not billed), and the cache rule. see [pdf](https://crawlbrulee.com/docs/scrape/pdf). also covered in `crawlbrulee-api`, `crawlbrulee-screenshots` and the overview in `crawlbrulee`.
- **dark-mode screenshots: `extract.screenshot.color_scheme`.** `dark` asks the site for its dark theme; `light` is the default. no extra credits. a pdf in the same request shows the dark theme too. `crawlbrulee-screenshots` explains it.
- **map: sitemaps one at a time, and what changed since a date.** every map now takes `sitemap_url` (read one sitemap file, on the same site or on a host the site's robots.txt names), `depth` (how many levels of nested sitemaps to read, `0` to `3`, default `3`), `entries` (`pages`, `sitemaps` or `all`) and `modified_since` (only entries whose `lastmod` is at or after a date), and every link is `{ url, type, lastmod }`. `crawlbrulee-map` teaches a plain map, listing a site's top sitemap files, reading one file, why a sitemap index at a low `depth` returns no pages, the three things to know about `modified_since`, and mapping a very large or deep site in several calls with `unread_sitemaps`. every skill now writes `max_results` and `total_before_max_results`, and `crawlbrulee-map` says which old names they replace and when those go. see [map](https://crawlbrulee.com/docs/map).
- the cli, mcp, js sdk and python sdk skills say how to send these and where the results land (`@crawlbrulee/sdk` and `crawlbrulee` 1.5.0, `@crawlbrulee/mcp` 1.5.0, cli 5.5.0).

### changed

- **the plugin connects to the hosted mcp server.** every plugin manifest now points at `https://api.crawlbrulee.com/mcp` instead of starting `@crawlbrulee/mcp` with `npx`. nothing runs on your machine, and the install no longer asks for an api key: the first time the agent uses the server, your host opens a crawlbrulee sign-in page in your browser, where you pick the organization to connect. the connection shows up with your other api tokens in the dashboard. want the local server instead? add `@crawlbrulee/mcp` yourself, as in [the mcp docs](https://crawlbrulee.com/docs/mcp).
- **two mcp skills.** `crawlbrulee-mcp` is now about the hosted server: connecting it with a browser sign-in in each host, its seven tools, and what it does differently (no `headers`, `cookies`, `raw_html`, screenshot `viewport` and `actions`, or `webhook`; `map` returns 200 links a page by default; `scrape` gives up after about 50 seconds; errors come back as the api's error object). the new `crawlbrulee-mcp-local` has everything about the local `@crawlbrulee/mcp` server: the api key setup per host, the extra fields and its error codes. `crawlbrulee` points to the hosted server for the mcp tools and lists both. the bundle now has eleven skills.
- `crawlbrulee-map` no longer says gzipped sitemaps (`.xml.gz`) can't be read: they are read on every map.
- the readme names pdf output, dark-mode screenshots and the new map options in its skill table.
- `crawlbrulee-screenshots`, `crawlbrulee-scrape`, `crawlbrulee-scrape-async`, `crawlbrulee-api` and both mcp skills say what happens to an async job that asks only for a screenshot or a pdf and gets none: it ends `failed`, not billed, and its result answers `job_failed`, or the same `422` as a sync scrape when the content type can't be captured.
- **markdown is the default.** a scrape with no `extract`, or with flags left out, returns `markdown` + `metadata`, not `cleaned_html` + `metadata`. `crawlbrulee-scrape`, the overview in `crawlbrulee`, `crawlbrulee-mcp` and `crawlbrulee-sdk-python` say so. naming other formats keeps the two defaults on, so every "only these values" example now sends `markdown: false` and `metadata: false`.
- `crawlbrulee-cli` no longer says the cli default differs from the api: both return markdown + metadata. it says that the flags you pick replace that default (cli 5.4.1), and the overview in `crawlbrulee` points there.
- `crawlbrulee-screenshots` says what a screenshot-only request is: every other format off, `markdown` and `metadata` included, and no `elements`.

## 1.4.0 (2026-11-06)

### added

- **`headers` and `cookies`.** two new scrape options that send your own http headers and cookies to the site, for example an `Authorization` header or a fixed `User-Agent` that a site has on its allowlist. they work on sync scrapes and async jobs, not on map. headers go only to the origin of `url` (or its https version, for an http url), cookies only to its host. a request with either never reads from or writes to the cache, and is billed as a normal fresh scrape. `crawlbrulee-scrape` has an example, the limits, the headers you can't set, and what to know about `User-Agent`, `Referer` and plain http urls. see [headers and cookies](https://crawlbrulee.com/docs/scrape/custom-headers-and-cookies). also covered in `crawlbrulee-api`, `crawlbrulee-scrape-async` (their values never come back in status or result answers), `crawlbrulee-map` (map takes neither) and the overview in `crawlbrulee`.
- **client names.** `headers` and `cookies` on `scrape()` / `scrapeAsync()` in the js sdk and on `scrape` / `scrape_async` in the python sdk (1.4.0), the same input on the `scrape` and `scrape_async` tools in the mcp (1.4.0), and the `--header "Name: value"`, `--cookie "name=value"`, `--headers <json>` and `--cookies <json>` flags in the cli (5.4.0). the mcp skill says that values pass through the chat, so use short-lived tokens there.
- **the `custom_user_agent` warning.** the request set its own `User-Agent` header; this can lower the success rate. added to the warning table in `crawlbrulee-scrape`.
- **`:scope` in `fields`.** `crawlbrulee-scrape` explains that the selector `:scope` reads the match itself, for example the matched element's own `href`; the cli, mcp and sdk skills say it in one line.

### changed

- the plugin starts `@crawlbrulee/mcp` 1.4.0.
- the elements examples use a made-up shop (`shop.example.com`, `article.book`, `.price`), instead of a scraping sandbox run by another company.

## 1.3.8 (2026-10-30)

### changed

- the screenshots skill says to use a screenshot link exactly as returned. if a link returns `403` or the image doesn't load, remove every query parameter except `token` and `expires`; if it still fails after the `expires` time, scrape the page again for a new link.

## 1.3.7 (2026-10-26)

### changed

- the setup skills say who handles the api key: your user gets it (sign up if needed, no credit card, then **tokens** → **create token** in the dashboard) and puts it in place themselves, never through the chat. the agent guides them and checks the key with `whoami`.
- the mcp skill adds the server with no key in its config, and names the file and line for the hosts that need one (Codex, Gemini CLI, Cursor, Windsurf, Claude Desktop). the full entries are in the docs: https://crawlbrulee.com/docs/mcp#install.
- the skills say what to do when the key is missing, and when the api answers `invalid_credentials`: your user creates a new key and replaces the old one.
- every host starts the mcp server with `npx -y @crawlbrulee/mcp@1.3.3`, whose key errors say the same.

## 1.3.6 (2026-10-19)

### changed

- the OpenAI plugin manifest follows OpenAI's directory rules: three starter prompts instead of five, the support, privacy and terms links, "GDPR" in capitals, and a brand color with enough contrast against white (`#E0B000`). other hosts are not affected.

## 1.3.5 (2026-10-19)

### added

- the Claude plugin manifest names the links for plugin directory listings: the docs home, support (this repo's issues), the privacy policy and the terms. hosts don't read these fields; nothing changes in how the plugin works.

## 1.3.4 (2026-10-19)

### added

- the readme says what gets sent where: your requests go to `https://api.crawlbrulee.com`, screenshot links point to `cdn.crawlbrulee.com`, and results are deleted no later than 90 days after they're created. it links the privacy policy and the data processing agreement.

## 1.3.3 (2026-10-19)

### changed

- every host, Claude included, starts the mcp server the same way again: `npx -y @crawlbrulee/mcp@1.3.2`. the plugin no longer ships a `package.json` or `package-lock.json`. in 1.3.2, hosts that read the Claude plugin files but don't install a lockfile could not start the server.

## 1.3.2 (2026-10-19)

### changed

- in Claude, the plugin now installs the mcp server from a `package-lock.json` when you install the plugin, and starts it from there. every package version is fixed in the lockfile, and nothing runs during the install. other hosts still start it with `npx -y @crawlbrulee/mcp@1.3.2`.
- `crawlbrulee-api` no longer pre-approves every `curl` command. your agent asks before it runs one.

## 1.3.1 (2026-10-19)

### changed

- the plugin now starts one exact version of the mcp server, `@crawlbrulee/mcp@1.3.2`, instead of the newest one on npm. plugin stores review the server a plugin runs, so each plugin release names the version it was tested with. to use a newer mcp server before the next plugin release, set it up by hand as shown in `crawlbrulee-mcp`.
- shorter one-line description for the OpenAI plugin directory, which allows 30 characters.

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
