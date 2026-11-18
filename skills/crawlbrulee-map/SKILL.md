---
name: crawlbrulee-map
description: use when discovering which urls exist on a site with crawlbrulee before scraping them — sitemap plus homepage link discovery, filtering to internal, external, or subdomain links, pagination through large result sets, and reading truncation info. also covers listing a site's sitemap files, reading one sitemap file at a time (sitemap_url, depth, entries), finding what changed since a date (modified_since, lastmod), and mapping a very large or deep site in several calls with unread_sitemaps. this is how you plan a crawl, since there is no crawl endpoint.
license: Apache-2.0
metadata:
  author: crawlbrulee
  homepage: https://crawlbrulee.com
---

# 🍮 crawlbrulee map

map enumerates the urls on a site. it reads the site's sitemaps and the links on its home page, dedupes the result, and paginates it. every link is `{ url, type, lastmod }`: a page or a sitemap file, with the date the sitemap gives. use it to plan what to scrape.

new to crawlbrulee? read the **crawlbrulee** skill first. for the shared bits — auth, proxies, caching, errors — read **crawlbrulee-api**.

## there is no crawl endpoint

map is half of the answer to "crawl this site". the other half is scraping each url you keep. the loop is yours to write:

1. **map** the site to get candidate urls, paginating until you have enough.
2. **filter** the list yourself — keep a path prefix, drop assets and anchors, cap the count.
3. **scrape** each survivor with the formats you want (see **crawlbrulee-scrape**).
4. **assemble** the results.

step 2 is the one that matters. map returns everything it finds; you decide what's worth scraping. scraping is where the cost is, so cap the list before you start — check `usage` first, and see <https://crawlbrulee.com/pricing> for the cost model.

## the request

```bash
curl -X POST https://api.crawlbrulee.com/api/map \
  -H "Authorization: Bearer $CRAWLBRULEE_API_KEY" -H "Content-Type: application/json" \
  -d '{ "url": "https://example.com", "limit": 1000, "page": 1 }'
```

| field | type / values | default | notes |
| --- | --- | --- | --- |
| `url` | string | — | any url on the site — see normalization below |
| `sitemap_url` | full http or https address | — | start from this sitemap file. it must be on the same site as `url`, or on a host the site's robots.txt names (a CDN, say); a file on another host that robots.txt does not name is a `400`. to read a file on another site, map that site. left out: start from the site's top sitemap files |
| `depth` | integer, `0` to `3` | `3` | how many levels of nested sitemaps to read, see [levels](#how-a-map-reads-sitemaps). a higher value gets a `400` `validation_error` |
| `entries` | `pages` · `sitemaps` · `all` | `pages` | what to return: page addresses, sitemap files, or both |
| `modified_since` | a date (`2026-11-01`, the start of that day in utc) or an iso-8601 datetime with a zone | — | only entries whose `lastmod` is at or after it. see [what changed](#what-changed-since-a-date) |
| `max_results` | integer | `5000` | the most entries the map finds, counting only the kind in `entries`, max `100000` — discovery **stops** here |
| `sitemap_only` | boolean | `false` | leave out the links found on the home page |
| `types.internal` | boolean | `true` | same-domain links |
| `types.internal_subdomains` | boolean | `true` | links on subdomains of the same site |
| `types.external` | boolean | `true` | links pointing off-site |
| `page` | integer | `1` | 1-indexed |
| `limit` | integer | `5000` | urls per page, max `10000` |
| `proxy` | `basic` · `advanced` · `auto` | `auto` | see **crawlbrulee-api** |
| `cache.max_age` | seconds or iso-8601 datetime | see docs | `0` forces a fresh map |
| `location.country` | string | — | alpha-2, or `eu` / `europe` |
| `zero_data_retention` | boolean | `false` | keep the result out of the shared cache, +1 credit — see below |

**renamed fields.** `max_results` was called `max_urls`, and `response_meta.truncation.total_before_max_results` was `total_before_max_urls`. the old names still work and mean the same (when both are sent, only `max_results` is used), but they are removed on **2027-02-01**: write the new ones. `discovery_cap_reason` already says `"max_results"` where it said `"max_urls"`.

**`zero_data_retention: true`** (top level, not inside `cache`) keeps the result out of the shared cache; anything stored to deliver it is kept for 24 hours, then deleted. it adds 1 credit and must be enabled for your organization. see [zero data retention](https://crawlbrulee.com/docs/zero-data-retention).

two differences from scrape worth remembering: **map takes `location.country` but not `location.locale`**, and it has no `require_js`.

**map takes no `headers` or `cookies`**: it sees a site the way a visitor without them does. if the pages you want need your own headers or cookies, list their urls another way and scrape each one with `headers` or `cookies` (see **crawlbrulee-scrape**).

a `depth` above `3`, an unknown `entries`, or a `modified_since` date and time with no time zone is a `400` `validation_error`. a bad `sitemap_url` is a `400` like a bad `url`. neither is billed.

### `limit` and `max_results` are not the same ceiling

`limit` is a page size. it decides how many of the collected urls come back in *this* response; you page for the rest, and nothing is lost.

`max_results` is the collection ceiling. discovery stops the moment it fills up, so urls past it are never collected and **no amount of paging brings them back**. both default to `5000`.

so a site with 40 000 pages needs `max_results: 40000` *and* paging. a site bigger or deeper than one map holds needs several calls: see [below](#a-very-large-or-deep-site-in-several-calls). and a cached map built under a smaller `max_results` is not reused for a bigger request: asking for more re-runs discovery and is billed as a fresh map.

## how a map reads sitemaps

a site's sitemaps form a tree: a sitemap file either lists pages, or it is a **sitemap index** that lists more sitemap files and no pages. map reads it level by level, and `depth` says how far down it goes.

```text
level 0   the list of the site's top sitemap files (its robots.txt and the usual addresses)
level 1   https://example.com/sitemap_index.xml              index of 2 files
level 2     https://example.com/sitemap-pages.xml            pages
level 2     https://example.com/sitemap-products.xml         index of 3 files
level 3       https://example.com/sitemaps/products-1.xml.gz   pages
```

- a file at level N is read only when N ≤ `depth`. the default `depth: 3` reads the top files and two nested levels. that is a **plain map** (`{ "url": … }`), which returns pages.
- **with `sitemap_url`, level 0 is that file, and it is read.** without it, level 0 is only the list of top files, which `depth: 0` returns without reading them.
- a sitemap file below `depth` is not read. it lands in `response_meta.truncation.unread_sitemaps`, and `discovery_cap_reason` is `"depth"`. that is the answer to the depth you asked for, **not a fault**: read those files with `sitemap_url` in more calls.
- `max_results` counts only the kind in `entries`.

### 1. a plain map

```json
{ "url": "https://example.com" }
```

returns pages from the sitemaps and the home page. a home page link is `{ "type": "page", "lastmod": null }`.

### 2. list a site's top sitemap files

```json
{ "url": "https://example.com", "depth": 0, "entries": "sitemaps" }
```

```jsonc
"links": [
  { "url": "https://example.com/sitemap_index.xml", "type": "sitemap", "lastmod": null },
  { "url": "https://example.com/sitemap-news.xml", "type": "sitemap", "lastmod": null }
]
// truncation: discovery_cap_reason "depth", and both files in unread_sitemaps: expected, they weren't read
```

top files have no `lastmod`.

### 3. read one sitemap file

```json
{ "url": "https://example.com", "sitemap_url": "https://example.com/sitemap-pages.xml" }
```

`url` still names the site: links are classed internal or external against it, and `types` filters pages (never sitemap files). a map with `sitemap_url` never adds home page links.

### why a sitemap index can return no pages

a sitemap index holds only sitemap files. read one at a low `depth` with the default `entries: "pages"`, and `links` is empty:

```json
{ "url": "https://example.com", "sitemap_url": "https://example.com/sitemap-products.xml", "depth": 0 }
```

the answer doesn't say "this was an index": an empty `links`, `discovery_cap_reason: "depth"` and the index's files in `unread_sitemaps` are the sign. ask again with `entries: "sitemaps"` (or `"all"`) to get those files as links, or with a higher `depth` to read them. a plain map of a site whose pages sit more than three levels down does the same.

### what changed since a date

```json
{ "url": "https://example.com", "sitemap_url": "https://example.com/sitemap-products.xml", "modified_since": "2026-11-01T00:00:00Z" }
```

returns only entries whose `lastmod` is at or after the date. three things to know:

1. **an entry with no `lastmod` is left out**, so every entry returned matches. home page links have no `lastmod`, so none comes back; a site whose sitemaps give no dates returns nothing.
2. **a nested sitemap whose own `lastmod` is older is not read.** a site that doesn't update the dates in its index when a page changes can hide changed pages this way. if you don't trust its dates, map without `modified_since` and compare `lastmod` yourself.
3. **it always runs fresh**: it never reads from or adds to the cache, so `cache.max_age` does nothing and every call, every `page` too, is billed as a fresh map. set a large `limit`.

### a very large or deep site in several calls

one call stops at `max_results` (at most `100000`), at its own time and file limits, and at its `depth` (at most `3`). for anything bigger, one sitemap file per call:

1. list the top files: `{ "url": …, "depth": 0, "entries": "sitemaps" }`.
2. read each with `{ "url": …, "sitemap_url": <file>, "max_results": 100000 }`, paging with `page` while `has_more` is true.
3. add every address in that answer's `unread_sitemaps` to your list, and read those too.
4. read each file once, and keep the ones that stay unread to try again later.

`unread_sitemaps` holds up to 1000 addresses. when it is full, ask the same thing with `entries: "sitemaps"`: you get every sitemap file the call found as links, paged like any answer.

docs, with the loop in js and python: [map a very large or deep site](https://crawlbrulee.com/docs/map/sitemaps#map-a-very-large-or-deep-site-in-several-calls).

## url normalization

**map always targets the site root.** the query string, fragment, and path are dropped, and `www.` is collapsed — so `https://example.com/blog/post?x=1` maps `example.com`. subdomains are preserved, so `https://docs.example.com/guide` maps `docs.example.com`, which is a different site from `example.com`.

mapping five different pages of the same site, with the same `sitemap_url` and `depth`, is one map, served from cache after the first.

### the urls you get back are a different form

returned links are **not** reduced to the root the way the mapped url is — each one comes back normalized.

this is the same url form `/scrape` reports for the page it fetched, so map-then-scrape stays on one host and the two responses agree on the string. mapping `https://www.example.com` therefore gives you `https://www.example.com/…` links even though the map itself is keyed on `example.com`. the two host forms of one page are still deduped to a single entry.

## the response

```jsonc
{
  "links": [{ "url": "https://example.com/about", "type": "page", "lastmod": "2026-11-12T00:00:00.000Z" }],
  "response_meta": {
    "pagination": { "page": 1, "limit": 1000, "total": 4231, "total_pages": 5, "has_more": true },
    "truncation": {
      "storage_capped": false,
      "response_capped": false,
      "total_before_max_results": 4231,
      "total_detected_before_storage_cap": 4231,
      "discovery_capped": false,
      "discovery_cap_reason": null,
      "sitemaps_skipped": 0,
      "unread_sitemaps": []
    },
    "usage": {
      "total_credit_cost": 1, "engine_credit_cost": 1, "proxy_multiplier": 1,
      "zero_data_retention_credit_cost": 0,
      "engine": "http", "proxy": "basic"
    }
  }
}
```

every link has all three fields:

- **`type`**: `page`, or `sitemap` for a sitemap file you can read with `sitemap_url` in another call (a file on another site only when the site's robots.txt names its host; otherwise map that file's own site). a home page link is a `page`.
- **`lastmod`**: the entry's `lastmod` from the sitemap, an iso-8601 date and time in utc, or `null` when the sitemap gives none or it can't be read. always `null` for a home page link.

if you need a link's anchor text, that's the scrape response's `links` field, not this. the site root is listed only when a sitemap lists it.

**ordering is stable across pages**, so paginating won't shuffle or drop entries under you. the most useful links come first.

`links[]` has no `source` field — don't expect one, and don't infer it from position.

**page 1 is the useful page** — if you only want a site's structure, one page at a small `limit` is usually enough.

### when the site answers with errors

map has **no `page_status_code`**. a map reads many things — sitemap files and the home page — so no single status would describe it.

- **a map that found urls is a `200` with those urls**, even if the home page itself was a `404`.
- **if the site answered only with statuses we don't bill (a `5xx`, for example), or not at all, you get a `200` with an empty `links` list**, and it costs 0. an empty map from a site that answered `404` is billed like any map, because a `404` is a real answer. an empty list is not proof the site has no pages — scrape the home page to see what the site is answering.
- **a `429` from the site** on the file in `sitemap_url`, on its robots.txt, or on one of its top sitemap files ends the map with `429` `too_many_requests` (`limited_by: "target_site"`, with a `Retry-After` header when the site sent one), not billed. wait, then ask again. a `429` on a nested file further down only lands that file in `unread_sitemaps`.
- in rare cases map answers `target_unreachable` (502): we could not reach the site at all. it is not billed; retry later, and check the url if it keeps happening.

### pagination

request `page: 1`, then keep going while `response_meta.pagination.has_more` is true:

```jsonc
"pagination": { "page": 1, "limit": 1000, "total": 4231, "total_pages": 5, "has_more": true }
```

`total` is the full count across all pages (after `entries`, `types` and `max_results`), not the length of this one.

### truncation

`truncation` tells you whether you're seeing everything the site has. it has two halves, and neither one is about `limit` or pagination — paging never loses urls, it just spreads the same stored list across pages.

**the stored list itself was cut shorter than what's really on the site:**

- **`response_capped`** — more links were eligible than `max_results`, so the list was trimmed. discovery itself *stops* at `max_results`, so this is normally `true` only when home-page links pushed the total past it. it has nothing to do with `limit` or pagination.
- **`storage_capped`** / **`total_detected_before_storage_cap`** — the site had more urls than one map holds (100 000).
- **`total_before_max_results`** — the count found before `max_results` applied. ⚠️ since discovery *stops* at `max_results`, this normally just equals `max_results` on a big site. it is **not** "how many the site has".

**discovery gave up before it finished, so it never even found everything** — a map cut short here reports `response_capped: false`, since nothing was left over to trim:

- **`discovery_capped`** (boolean) — sitemap discovery stopped before reading every sitemap file it found. when true, the site has more than this map lists.
- **`discovery_cap_reason`** — which limit stopped it first, or `null` when nothing did: `max_results` · `depth` · `time` · `file_budget` · `file_size` · `unread_files`.
- **`sitemaps_skipped`** (integer ≥ 0) — how many sitemap files were skipped or only partly read.
- **`unread_sitemaps`** (list of urls) — the sitemap files this call knew about but didn't read whole. read each with `sitemap_url` in another call, except a file that says it is over the size limit: no call reads that one.

#### reading `discovery_cap_reason`

| reason | what happened | what to do |
| --- | --- | --- |
| `null` | nothing capped discovery | the list is complete — stop |
| `max_results` | your ceiling filled up while sitemap files were still unread | **retry with a higher `max_results`** |
| `depth` | sitemap files lie below the `depth` you asked for | expected when you asked for a low `depth`. read the files in `unread_sitemaps` with `sitemap_url`, or raise `depth` |
| `unread_files` | a sitemap file couldn't be read whole this time: a `429`/`5xx`, a file that is not a readable sitemap, or a download that broke off (the links received before the break come back, and the file is in `unread_sitemaps`) | **a retry may help**: it's often transient, so a retry soon can work. if it keeps happening, the site is publishing something we can't read: stop retrying |
| `time` · `file_budget` | the site is bigger or slower than one discovery pass | raising `max_results` won't help. read the files in `unread_sitemaps` with `sitemap_url`, one call each |
| `file_size` | a sitemap file was too large. a file that says up front that it is over the size limit is not downloaded: nothing comes back, it is in `unread_sitemaps`, and no call reads it. a file of pages that turns out too large while it is read gives the links in its first part. any other file that turns out too large, such as an index, gives nothing and is in `unread_sitemaps` | a retry reads the same part, so accept a partial list for that file |

⚠️ **a full page is not a signal, and neither is `response_capped`.** with the default `max_results` of 5000, a large site returns exactly 5000 links with `response_capped: false` — because discovery stopped right on the cap, nothing was trimmed on the way out. that looks complete and isn't. `discovery_cap_reason: "max_results"` is the honest signal, and a retry with a bigger `max_results` is the fix.

## tips

- **check `discovery_cap_reason` before you trust the list.** it is the one field that distinguishes "this is the whole site" from "this is the first 5000 urls of it".
- **`sitemap_only: true`** leaves the home page links out when you only want what the sitemaps say.
- **filter to a section** with a path prefix before scraping — mapping a docs site and keeping only `/guide/` is the common shape.
- **map is cache-friendly.** repeat maps of the same root are served from cache within the freshness window; pass `cache.max_age: 0` when you specifically need a fresh inventory. `modified_since` never uses the cache.
- **map costs credits** — a fresh map reports `engine: "http"`; a cached map reports `engine: "cache"`. those are the only map engine values. the returned `proxy` is the resolved `basic` or `advanced` tier, never `auto`. read `response_meta.usage.total_credit_cost` for what it cost (`total_credit_cost = engine_credit_cost × proxy_multiplier + zero_data_retention_credit_cost`). see <https://crawlbrulee.com/pricing>. don't assume it's free.

## see also

- scrape the urls you found: **crawlbrulee-scrape** · shared contract: **crawlbrulee-api**
- drive it from: **crawlbrulee-cli**, **crawlbrulee-mcp**, **crawlbrulee-mcp-local**, **crawlbrulee-sdk-js**, **crawlbrulee-sdk-python**
- docs: <https://crawlbrulee.com/docs/map>
