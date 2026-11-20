---
name: crawlbrulee-cli
description: use when driving crawlbrulee from a terminal or a shell script — the `crawlbrulee` / `npx crawlbrulee` command. covers scrape url, scrape status/result/wait, map, usage, whoami, login/logout/view-config, every flag, picking values by css selector (--element / --elements), your own headers and cookies (--header / --cookie), the screenshot shorthand, and text-vs-json output.
license: Apache-2.0
metadata:
  author: crawlbrulee
  homepage: https://crawlbrulee.com
---

# 🍮 crawlbrulee cli

the `crawlbrulee` command scrapes pages, maps sites, and inspects your account from the shell. it's `npx`-runnable with zero install and wraps `@crawlbrulee/sdk`. output is tty-aware: readable text in a terminal, json when piped or written to a file.

this skill is the command surface. for what the api actually does — extract formats, proxy tiers, caching, errors — read **crawlbrulee-api** and the capability skills.

## setup

```bash
npx crawlbrulee scrape url https://example.com   # one-off, no install
npm install -g crawlbrulee                       # or install once, then `crawlbrulee …`
```

authenticate one of three ways, checked in this order per request:

1. `--api-key cwbl_…` on the call
2. the `CRAWLBRULEE_API_KEY` environment variable
3. a stored config written by `crawlbrulee login`

```bash
crawlbrulee login                    # prompts for the key, input hidden
crawlbrulee login --api-key cwbl_…   # non-interactive
crawlbrulee view-config              # show resolved config, key masked
crawlbrulee logout                   # remove stored credentials
```

`crawlbrulee <command> --help` is the complete flag list.

## `crawlbrulee scrape url <url>`

note the shape: **`scrape` is a command group, so scraping a url is `scrape url <url>`** — not `crawlbrulee scrape <url>`.

**the cli defaults to markdown + metadata**, the same as the api. the flags you pick replace that default: you get what you named plus metadata, so pass every format you want (`-l` alone gives links + metadata, no page body). `--no-metadata` drops the metadata.

| flag | short | effect |
| --- | --- | --- |
| `--markdown` | `-m` | markdown |
| `--cleaned-html` | `-c` | main-content html |
| `--raw-html` | `-r` | raw html |
| `--links` | `-l` | all links |
| `--images` | `-i` | inline images |
| `--screenshot` | `-ss` | capture a screenshot (syntax below) |
| `--element <name=selector>` | | a named value by css selector, the text of the first match; repeatable (cli `5.3.0`+) |
| `--elements <json>` | | named values in the full json form; `@path` reads a file (cli `5.3.0`+) |
| `--all` | | every extract field at once |
| `--no-metadata` | | drop page metadata |
| `--require-js` | | render JavaScript |
| `--proxy <tier>` | | `basic` · `advanced` · `auto` |
| `--exclude-selectors "nav,footer"` | | css selectors stripped from the result |
| `--cache-max-age <seconds>` | | freshness cutoff; `0` forces a fresh fetch |
| `--locale <bcp47>` | | e.g. `en-US` |
| `--country <iso>` | | e.g. `US`, or `eu` / `europe` |
| `--zero-data-retention` | | keep the result out of the shared cache, +1 credit (cli `5.2.0`+) |
| `--header "Name: value"` | | an http header for the site; repeatable (cli `5.4.0`+) |
| `--cookie "name=value"` | | a cookie for the site; repeatable (cli `5.4.0`+) |
| `--headers <json>` | | headers as a json object; `@path` reads a file (cli `5.4.0`+) |
| `--cookies <json>` | | cookies as json; `@path` reads a file (cli `5.4.0`+) |
| `-o, --output <file>` | | write to a file instead of stdout |

```bash
crawlbrulee scrape url https://example.com                 # markdown
crawlbrulee scrape url https://example.com --all           # everything
crawlbrulee scrape url https://example.com -c              # cleaned html
crawlbrulee scrape url https://example.com --proxy advanced --require-js
crawlbrulee scrape url https://example.com | jq .links     # json when piped
crawlbrulee scrape url https://example.com --json | jq .response_meta.usage
```

`--proxy` takes `basic`, `advanced`, or `auto` only. omit it and the server applies its default.

`--zero-data-retention` (also on `scrape url --async` and on `map`) keeps the result out of the shared cache; anything stored to deliver it is kept for 24 hours, then deleted. it adds 1 credit and must be enabled for your organization. see [zero data retention](https://crawlbrulee.com/docs/zero-data-retention). an older cli fails with `unknown option`; check `crawlbrulee --version`.

### your own headers and cookies

send your own http headers and cookies with the scrape, for example an `Authorization` header or a fixed `User-Agent` that a site has on its allowlist:

```bash
crawlbrulee scrape url https://app.example.com/orders \
  --header "Authorization: Bearer $APP_TOKEN" \
  --cookie "session_id=$APP_SESSION"

crawlbrulee scrape url https://app.example.com/orders --headers @headers.json --cookies @cookies.json
```

- `--header` and `--cookie` can be given more than once. `--headers` takes a json object of name → value; `--cookies` takes the api's `cookies` json, a list of `{ "name", "value" }`.
- for the same field, use either the flag or the json form, not both: `--header` with `--headers` is refused, and so is `--cookie` with `--cookies`.
- both work with `--async`, not on `map`.
- keep secrets out of the command itself. `@path` (`--headers @headers.json`, `--cookies @cookies.json`) is the safest form: the value never appears in the command. an environment variable, as above, keeps it out of your shell history, but the shell puts the real value into the command, where other users of the machine can see it with `ps`.
- headers go only to the origin of the url (or its https version, for an http url), cookies only to its host. the request skips the cache and is billed as a normal fresh scrape. a header you can't set fails with `validation_error`, and the message names it. your own `User-Agent` adds the `custom_user_agent` warning. see **crawlbrulee-scrape** and [headers and cookies](https://crawlbrulee.com/docs/scrape/custom-headers-and-cookies).
- an older cli fails with `unknown option`; check `crawlbrulee --version`.

### picking values: `--element` and `--elements`

when you need a few values (a price, a title, every product on a list), name them by css selector and read them from `elements` in the json. far less to read than the markdown, and no extra credits.

```bash
# the text of the first match, one flag per name
crawlbrulee scrape url https://shop.example.com/books/ --element heading=h1 --element price=.price --json | jq .elements

# the full form: one object per product, with fields read inside each product
# (the selector :scope reads the product element itself)
crawlbrulee scrape url https://shop.example.com/books/ --json --elements '{
  "books": { "selector": "article.book", "all": true,
    "fields": { "title": { "selector": "h3 a", "output": "attribute", "attribute": "title" }, "price": ".price" } }
}' | jq '.elements.books[:2]'
# → [{ "title": "The Night Garden", "price": "€18.90" }, { "title": "Salt and Stone", "price": "€22.50" }]

crawlbrulee scrape url https://shop.example.com/books/ --elements @elements.json --json   # read the json from a file
```

- on their own, `--element` / `--elements` return only the elements (plus metadata unless `--no-metadata`); add `-m` or another content flag to get the page too.
- `--element` splits on the **first** `=`, so `--element 'link=a[href="/cart"]'` works. always put single quotes around a selector that has anything beyond letters, digits, `.`, `#`, `-` or `_` — `--element 'item=li:not(.ad)'` — or the shell changes it before the cli sees it.
- `--elements` takes the same json as `extract.elements` in the api — `selector`, `output`, `attribute`, `all`, `fields`. see **crawlbrulee-scrape**.
- the two flags combine, and both work with `--async`.
- each name may appear only once across both flags, `--elements` may be given only once, and `--elements '{}'` (an empty object) is refused, even next to `--element`.
- a name with no match is `null` (`[]` for a list). an invalid selector fails with `validation_error`. when a value hits a limit, `warnings` has `elements_truncated`.
- an older cli fails with `unknown option`; check `crawlbrulee --version`. full rules: [targeted extraction](https://crawlbrulee.com/docs/scrape/targeted-extraction).

### screenshot shorthand (`-ss` / `--screenshot`)

positional and comma-separated, read strictly left to right. to set a later position you must give real values for every earlier one — empty slots are rejected, and a width without a height is an error. so a mobile or sliced shot has to include width and height.

```
-ss
-ss <mode>
-ss <mode>,<width>,<height>
-ss <mode>,<width>,<height>,<device>
-ss <mode>,<width>,<height>,<device>,<slice-height>
```

| position | values | default |
| --- | --- | --- |
| 1 mode | `viewport` · `full` · `full_page` | `full_page` |
| 2 width | integer 16–10000 | server default |
| 3 height | integer 16–10000 | server default |
| 4 device | `desktop` · `mobile` | server default |
| 5 slice-height | integer ≥ 500 | none (no tiling) |

`full` is shorthand for `full_page`. a width or height outside `16–10000` is rejected up front rather than round-tripping to a server `400`.

```bash
crawlbrulee scrape url https://x.com -ss viewport
crawlbrulee scrape url https://x.com -ss full,1920,1080,mobile
crawlbrulee scrape url https://x.com -ss full,1280,720,desktop,800   # sliced into tiles
```

see **crawlbrulee-screenshots** for what the options mean and what comes back.

### a pdf, and dark-mode screenshots

cli `5.5.0` adds the api's `extract.pdf` (the page saved as a pdf, see **crawlbrulee-scrape**) and `extract.screenshot.color_scheme` (`light` or `dark`, see **crawlbrulee-screenshots**). read the pdf from the json: `--json | jq -r .pdf.url` prints its signed link, and `.pdf.bytes` its size. the link expires 24 hours after the scrape, so download the file right away (`curl -o page.pdf "$(… --json | jq -r .pdf.url)"`), and use the link exactly as printed. a pdf costs what a screenshot costs. when no pdf came back, `warnings` has `pdf_unavailable`; a pdf-only request fails with `unsupported_pdf_output` when the content type can't be made into a pdf, not billed.

## async jobs

submit in the background instead of holding the connection open — good for heavy js rendering or long-page screenshots. `--async` prints a `job_id` and exits; add `--wait` to poll to completion and print the result in one command.

```
--async                     submit a background job (prints the job_id)
--wait                      with --async, poll until done and print the result
--interval <seconds>        seconds between status polls while waiting
--timeout <seconds>         max seconds to wait (0 = wait forever)
--webhook-url <url>         completion webhook endpoint (requires --async)
--webhook-metadata <json>   json object echoed back in the webhook (requires --webhook-url)
```

```bash
crawlbrulee scrape url https://example.com --async                     # prints `job_id: <id>`
crawlbrulee scrape url https://example.com --async --json | jq -r .job_id
crawlbrulee scrape url https://example.com --async --wait
crawlbrulee scrape url https://example.com --async \
  --webhook-url https://hooks.example.com/cb \
  --webhook-metadata '{"order":"abc"}'
```

three subcommands act on a `job_id`:

```bash
crawlbrulee scrape status <job-id>    # pending | running | done | failed (+ usage when done)
crawlbrulee scrape result <job-id>    # fetch the finished result (errors if not done)
crawlbrulee scrape wait   <job-id>    # poll until done, then print the result
```

`--interval` and `--timeout` are in **seconds**, and apply to `scrape wait` and `--async --wait`. constraints: `--wait` requires `--async`; `--interval`/`--timeout` require `--wait`; `--webhook-url`/`--webhook-metadata` require `--async`, and `--webhook-metadata` requires `--webhook-url` and must be a json **object**. while waiting in a terminal the cli prints a one-line note to stderr so stdout stays pipeable; Ctrl-C cancels.

see **crawlbrulee-scrape-async** for the lifecycle and webhook verification.

## `crawlbrulee map <url>`

```bash
crawlbrulee map https://example.com --limit 500 --page 2
crawlbrulee map https://example.com --sitemap-only
crawlbrulee map https://example.com --text > urls.txt   # one url per line
```

| flag | effect |
| --- | --- |
| `--limit <n>` | page size — how many of the collected urls come back in **this** response, default `5000`, max `10000`. it is not how many urls get collected |
| `--max-results <n>` | total entries to collect, counting only the kind in `--entries`, default `5000`, api max `100000` |
| `--sitemap-url <url>` | read this one sitemap file (on the same site, or on a host the site's robots.txt names) instead of starting from the site's top sitemap files |
| `--depth <n>` | levels of nested sitemaps to read, `0` to `3`, default `3`. `0` without `--sitemap-url` lists the site's top sitemap files without reading their entries, and `1` with `--entries sitemaps` lists them and every file they list |
| `--entries <kind>` | `pages` (default) · `sitemaps` · `all` |
| `--modified-since <date>` | only entries whose `lastmod` is at or after this date (`2026-11-01`, or a date and time with a zone). never served from the cache |
| `--page <n>` | page number (1-indexed) |
| `--sitemap-only` | use the sitemaps only, leave out the links found on the home page |
| `--internal-only` | same-domain links only |
| `--external-only` | external-domain links only |
| `--no-subdomains` | exclude subdomains from internal results |
| `--proxy <tier>` | `basic` · `advanced` · `auto` |
| `--cache-max-age <seconds>` | freshness cutoff |
| `--country <iso>` | proxy egress country |
| `--zero-data-retention` | keep the result out of the shared cache, +1 credit (cli `5.2.0`+) |
| `-o, --output <file>` | write to a file instead of stdout |

`--internal-only` and `--external-only` can't be combined. **there's no `--locale` on map** — it's a scrape-only option.

### a map collects at most `--max-results` entries

`--limit` only trims *this response*; page for the rest and nothing is lost. `--max-results` is the collection ceiling — discovery **stops** there, so urls past it are never collected and no amount of paging brings them back. both default to `5000`.

`--max-results` needs cli `5.5.0`; on an older cli use `--max-urls`.

raising `--max-results` doesn't cost extra credits by itself. but a cached map built under a smaller budget can't answer a bigger request, so the retry re-runs discovery and is billed as a fresh map.

**a capped map looks complete.** with the ceiling at 5000, a big site returns exactly 5000 urls. read the json to be sure — `crawlbrulee map <url> --json | jq .response_meta.truncation`. `discovery_cap_reason` is the field: `null` means the list is whole, `"max_results"` is the one reason worth retrying with a bigger ceiling, `"depth"` means sitemap files lie below `--depth` (read them with `--sitemap-url`), `"unread_files"` means some sitemap files couldn't be read whole this time (often transient, so a retry soon may return more; but if it keeps happening the site is publishing something we can't read), and `time` · `file_budget` · `file_size` mean a bigger ceiling changes nothing. see **crawlbrulee-map**.

### sitemaps one at a time (cli `5.5.0`)

```bash
crawlbrulee map https://example.com --depth 1 --entries sitemaps        # the top sitemap files and every file they list
crawlbrulee map https://example.com --sitemap-url https://example.com/sitemap-products.xml
crawlbrulee map https://example.com --modified-since 2026-11-01         # only what changed since then
```

in text mode a plain map stays one url per line, so it pipes straight into the next command. with `--entries all`, where pages and sitemap files mix, each line is `<type> <lastmod> <url>` (`-` for no date). when a call left sitemap files unread, the text ends with them as `#` comment lines: try each one next with `--sitemap-url`. in json every link is `{ url, type, lastmod }`:

```bash
# sitemap files to read next: each link of type sitemap, plus each sitemap_index link
# that is also in unread_sitemaps (an index this map read only in part)
crawlbrulee map https://example.com --depth 1 --entries sitemaps --json \
  | jq -r '.response_meta.truncation.unread_sitemaps as $u | .links[]
      | select(.type == "sitemap" or (.type == "sitemap_index" and (.url | IN($u[]))))
      | .url'
crawlbrulee map https://example.com --json | jq -r '.response_meta.truncation.unread_sitemaps[]'
```

a sitemap index holds only sitemap files, so reading one at a low `--depth` with the default `--entries pages` returns no pages: use `--entries sitemaps` or a higher `--depth`. `--modified-since` leaves out every entry with no `lastmod`. it reads every sitemap file whatever its own date and judges each entry by its own `lastmod`, so with `--entries sitemaps` a file comes back by its own date: use `--entries pages` to find the pages that changed. every call (every `--page` too) is billed as a fresh map. a `429` from the site on the file in `--sitemap-url` or on a top sitemap file ends the map with `too_many_requests`, not billed: wait, then run it again. the whole several-calls flow is in **crawlbrulee-map**.

## `usage` and `whoami`

```bash
crawlbrulee usage                              # credits, quota, concurrency, reset time
crawlbrulee usage --json | jq .available_credits
crawlbrulee whoami                             # org name + masked token preview
```

check `usage` before a credit-heavy job.

## output convention

- terminal → **text**; piped, redirected, or `-o <file>` → **json**.
- force it with `--json` or `--text`; `--compact` gives one-line json.
- in text mode, a scrape prints a trailing `# usage: <n> credits · engine <engine> · proxy <tier> · slices <0|1>` comment — the same `response_meta.usage` you'd get in json. `<n>` is what the call cost (`total_credit_cost`). map prints the same comment without the slice field because map does not produce screenshots; its engine is `http` or `cache`, and its resolved proxy is `basic` or `advanced` (never `auto`). `engine cache` identifies a cache hit.
- in text mode, elements print as an `elements: {…}` block, after the page body if you asked for one. when the page can't give some of what you asked for (say `--element` on a json url), a `# unsupported: <fields>` line names them, e.g. `# unsupported: elements`; the rest still prints.
- in json, read `response_meta.usage.total_credit_cost` for the cost.
- errors go to stderr as `error: <name> — <message>`.
- exit code is `0` on success, `1` on any failure, so you can branch on it in scripts.

### a `404` page is a success

a page the site really served is a successful scrape, whatever its own status. a `404`, `410` or `503` page prints like any page, and the command **exits `0`** — the scrape worked. the site's status is in `page_status_code`:

```bash
crawlbrulee scrape url https://example.com/old --json | jq .page_status_code          # → 404
crawlbrulee scrape url https://example.com/old --json | jq -e '.page_status_code < 400' # exit 1 on an error page
```

from cli `5.1.0`, text mode adds the status to the end of the usage comment when it isn't 2xx: `# usage: 15 credits · engine browser · proxy advanced · slices 0 · page status 404`. older clis don't show it; use `--json` there. **check the status before you use the content in a script** — the markdown of a `404` page is the site's "not found" text. if the json has no `page_status_code`, the response came from before the field existed, and the page was served normally.

## common errors

```
error: too_many_requests — please slow down (retry after 12000ms)
error: usage_allocation_error — out of credits (reason: credit_limit)
error: antibot_blocked — protected page
error: too_many_redirects — Target site redirected the request too many times.
error: page_too_large — The page is too large or too complex to convert.
error: target_unreachable — Could not reach the target site. (retrying later may help)
error: zero_data_retention_not_enabled — zero_data_retention is not enabled for your organization. Contact us to turn it on.
error: invalid_url — not a valid URL
error: service_unavailable — service temporarily unavailable (temporary — safe to retry)
error: not logged in — run `crawlbrulee login` or set CRAWLBRULEE_API_KEY
```

empty page? use `--require-js` when content renders client-side; for general retrieval failures, `--proxy advanced` uses the higher-success tier. an `antibot_blocked` response means the target's bot protection blocked the request; it isn't a retry signal. neither is `too_many_redirects` — the target redirected in a loop. `page_too_large` means the page's html was too large to process; it is terminal, so don't run the same command again. `zero_data_retention_not_enabled` means the flag isn't turned on for your organization; nothing was billed, and running it again won't help — drop `--zero-data-retention` or email sales@crawlbrulee.com. `target_unreachable` means we could not reach the site at all — nothing was billed; run it again after a pause, and if it keeps failing, check the url and whether the site is up. a `404` page is not an error at all — see above. out of credits or hitting concurrency? check `crawlbrulee usage`. a `service_unavailable` is a 503 on our side rather than a key problem — wait a moment and run the same command again, don't re-login or rotate the key. the full error reference is in **crawlbrulee-api**.

## see also

- what the api does: **crawlbrulee-api**, **crawlbrulee-scrape**, **crawlbrulee-screenshots**, **crawlbrulee-scrape-async**, **crawlbrulee-map**
- the library underneath: **crawlbrulee-sdk-js**
- docs: <https://crawlbrulee.com/docs/cli>
