---
name: crawlbrulee-cli
description: use when driving crawlbrulee from a terminal or a shell script — the `crawlbrulee` / `npx crawlbrulee` command. covers scrape url, scrape status/result/wait, map, usage, whoami, login/logout/view-config, every flag, the screenshot shorthand, and text-vs-json output.
license: Apache-2.0
metadata:
  author: crawlbrulee
  homepage: https://crawlbrulee.com
allowed-tools:
  - Bash(crawlbrulee *)
  - Bash(npx crawlbrulee *)
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
| `--all` | | every extract field at once |
| `--no-metadata` | | drop page metadata |
| `--require-js` | | render JavaScript |
| `--proxy <tier>` | | `basic` · `advanced` · `auto` |
| `--exclude-selectors "nav,footer"` | | css selectors stripped from the result |
| `--cache-max-age <seconds>` | | freshness cutoff; `0` forces a fresh fetch |
| `--locale <bcp47>` | | e.g. `en-US` |
| `--country <iso>` | | e.g. `US`, or `eu` / `europe` |
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
- in text mode, a scrape prints a trailing `# usage: <credits> credits · engine <engine> · proxy <tier> · slices <0|1>` comment — the same `response_meta.usage` you'd get in json. map prints the same comment without the slice field because map does not produce screenshots; its engine is `http` or `cache`, and its resolved proxy is `basic` or `advanced` (never `auto`). `engine cache` identifies a cache hit.
- errors go to stderr as `error: <name> — <message>`.
- exit code is `0` on success, `1` on any failure, so you can branch on it in scripts.

## common errors

```
error: too_many_requests — please slow down (retry after 12000ms)
error: usage_allocation_error — out of credits (reason: credit_limit)
error: antibot_blocked — protected page
error: too_many_redirects — Target site redirected the request too many times.
error: page_too_large — The page is too large or too complex to convert.
error: invalid_url — not a valid URL
error: service_unavailable — service temporarily unavailable (temporary — safe to retry)
error: not logged in — run `crawlbrulee login` or set CRAWLBRULEE_API_KEY
```

empty page? use `--require-js` when content renders client-side; for general retrieval failures, `--proxy advanced` uses the higher-success tier. an `antibot_blocked` response means the target's bot protection blocked the request; it isn't a retry signal. neither is `too_many_redirects` — the target redirected in a loop. `page_too_large` means the page's html was too large to process; it is terminal, so don't run the same command again. out of credits or hitting concurrency? check `crawlbrulee usage`. a `service_unavailable` is a 503 on our side rather than a key problem — wait a moment and run the same command again, don't re-login or rotate the key. the full error reference is in **crawlbrulee-api**.

## see also

- what the api does: **crawlbrulee-api**, **crawlbrulee-scrape**, **crawlbrulee-screenshots**, **crawlbrulee-scrape-async**, **crawlbrulee-map**
- the library underneath: **crawlbrulee-sdk-js**
- docs: <https://crawlbrulee.com/docs/cli>
