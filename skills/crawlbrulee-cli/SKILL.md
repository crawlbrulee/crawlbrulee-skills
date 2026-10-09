---
name: crawlbrulee-cli
description: use when driving crawlbrulee from a terminal or a shell script — the `crawlbrulee` / `npx crawlbrulee` command. covers scrape url, scrape status/result/wait, map, usage, whoami, login/logout/view-config, every flag, picking values by css selector (--element / --elements), the screenshot shorthand, and text-vs-json output.
license: Apache-2.0
metadata:
  author: crawlbrulee
  homepage: https://crawlbrulee.com
allowed-tools: Bash(crawlbrulee:*) Bash(npx crawlbrulee:*)
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

**the cli defaults to markdown + metadata**, which differs from the api's own default of metadata + cleaned html. naming any extract flag replaces that default set, so pass every format you want.

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

### picking values: `--element` and `--elements`

when you need a few values (a price, a title, every product on a list), name them by css selector and read them from `elements` in the json. far less to read than the markdown, and no extra credits.

```bash
# the text of the first match, one flag per name
crawlbrulee scrape url https://books.toscrape.com --element heading=h1 --element price=.price_color --json | jq .elements

# the full form: one object per product card, with fields read inside each card
crawlbrulee scrape url https://books.toscrape.com --json --elements '{
  "books": { "selector": "article.product_pod", "all": true,
    "fields": { "title": { "selector": "h3 a", "output": "attribute", "attribute": "title" }, "price": ".price_color" } }
}' | jq '.elements.books[:2]'
# → [{ "title": "A Light in the Attic", "price": "£51.77" }, { "title": "Tipping the Velvet", "price": "£53.74" }]

crawlbrulee scrape url https://books.toscrape.com --elements @elements.json --json   # read the json from a file
```

- on their own, `--element` / `--elements` return only the elements (plus metadata unless `--no-metadata`); add `-m` or another content flag to get the page too.
- `--element` splits on the **first** `=`, so `--element 'link=a[href="/cart"]'` works. always put single quotes around a selector that has anything beyond letters, digits, `.`, `#`, `-` or `_` — `--element 'item=li:not(.ad)'` — or the shell changes it before the cli sees it.
- `--elements` takes the same json as `extract.elements` in the api — `selector`, `output`, `attribute`, `all`, `fields`. see **crawlbrulee-scrape**.
- the two flags combine, and both work with `--async`.
- each name may appear only once across both flags, `--elements` may be given only once, and `--elements '{}'` (an empty object) is refused, even next to `--element`.
- a name with no match is `null` (`[]` for a list). an invalid selector fails with `validation_error`. when a value hits a limit, `warnings` has `elements_truncated`.
- an older cli fails with `unknown option`; check `crawlbrulee --version`. full rules: [elements](https://crawlbrulee.com/docs/scrape/elements).

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
| `--max-urls <n>` | total urls to collect, default `5000`, api max `100000` |
| `--page <n>` | page number (1-indexed) |
| `--sitemap-only` | use `sitemap.xml` only, skip homepage extraction |
| `--internal-only` | same-domain links only |
| `--external-only` | external-domain links only |
| `--no-subdomains` | exclude subdomains from internal results |
| `--proxy <tier>` | `basic` · `advanced` · `auto` |
| `--cache-max-age <seconds>` | freshness cutoff |
| `--country <iso>` | proxy egress country |
| `--zero-data-retention` | keep the result out of the shared cache, +1 credit (cli `5.2.0`+) |
| `-o, --output <file>` | write to a file instead of stdout |

`--internal-only` and `--external-only` can't be combined. **there's no `--locale` on map** — it's a scrape-only option.

### a map collects at most `max_urls` urls

`--limit` only trims *this response*; page for the rest and nothing is lost. `max_urls` is the collection ceiling — discovery **stops** there, so urls past it are never collected and no amount of paging brings them back. both default to `5000`.

`--max-urls` needs cli `4.3.0` or newer. on an older cli every map tops out at 5000 urls and the flag fails with `unknown option`; check with `crawlbrulee --version` and upgrade.

raising `max_urls` doesn't cost extra credits by itself. but a cached map built under a smaller budget can't answer a bigger request, so the retry re-runs discovery and is billed as a fresh map.

**a capped map looks complete.** with the ceiling at 5000, a big site returns exactly 5000 urls and the text output says nothing about it. read the json to be sure — `crawlbrulee map <url> --json | jq .response_meta.truncation`. `discovery_cap_reason` is the field: `null` means the list is whole, `"max_urls"` is the one reason worth retrying with a bigger ceiling, `"unread_files"` means some sitemap files couldn't be read this time (often transient — a retry soon may return more, since a map with this reason is cached for only 15 minutes instead of the usual window — but if it keeps happening the site is publishing something we can't read), and `time` · `file_budget` · `depth` · `file_size` mean a bigger ceiling changes nothing. see **crawlbrulee-map**.

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
