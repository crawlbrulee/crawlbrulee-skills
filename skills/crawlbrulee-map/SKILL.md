---
name: crawlbrulee-map
description: use when discovering which urls exist on a site with crawlbrulee before scraping them — sitemap plus homepage link discovery, filtering to internal, external, or subdomain links, pagination through large result sets, and reading truncation info. this is how you plan a crawl, since there is no crawl endpoint.
---

# 🍮 crawlbrulee map

map enumerates the urls on a site. it combines sitemap discovery with homepage link extraction, dedupes the result, and paginates it. use it to plan what to scrape.

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
| `sitemap_only` | boolean | `false` | use `sitemap.xml` only, skip homepage extraction |
| `types.internal` | boolean | `true` | same-domain links |
| `types.internal_subdomains` | boolean | `true` | links on subdomains of the same site |
| `types.external` | boolean | `true` | links pointing off-site |
| `max_urls` | integer | `5000` | ceiling on how many urls we collect, max `100000` — discovery **stops** here |
| `page` | integer | `1` | 1-indexed |
| `limit` | integer | `5000` | urls per page, max `10000` |
| `proxy` | `basic` · `advanced` · `auto` | `auto` | see **crawlbrulee-api** |
| `cache.max_age` | seconds or iso-8601 datetime | see docs | `0` forces a fresh map |
| `location.country` | string | — | alpha-2, or `eu` / `europe` |

two differences from scrape worth remembering: **map takes `location.country` but not `location.locale`**, and it has no `require_js`.

### `limit` and `max_urls` are not the same ceiling

`limit` is a page size. it decides how many of the collected urls come back in *this* response; you page for the rest, and nothing is lost.

`max_urls` is the collection ceiling. discovery stops the moment it fills up, so urls past it are never collected and **no amount of paging brings them back**. both default to `5000`.

so a site with 40 000 pages needs `max_urls: 40000` *and* paging. and because a map is cached on the site root, a cached map that was built under a smaller `max_urls` is not reused for a bigger request — asking for more re-runs discovery and is billed as a fresh map.

## url normalization

**map always targets the site root.** the query string, fragment, and path are dropped, and `www.` is collapsed — so `https://example.com/blog/post?x=1` maps `example.com`. subdomains are preserved, so `https://docs.example.com/guide` maps `docs.example.com`, which is a different site from `example.com`.

the cache is keyed on the normalized root, which means mapping five different pages of the same site is one map, served from cache after the first.

### the urls you get back are a different form

returned links are **not** reduced to the root the way the mapped url is — each one comes back normalized.

this is the same url form `/scrape` reports for the page it fetched, so map-then-scrape stays on one host and the two responses agree on the string. mapping `https://www.example.com` therefore gives you `https://www.example.com/…` links even though the map itself is keyed on `example.com`. the two host forms of one page are still deduped to a single entry.

## the response

```jsonc
{
  "links": [{ "url": "https://example.com/about" }],
  "response_meta": {
    "pagination": { "page": 1, "limit": 1000, "total": 4231, "total_pages": 5, "has_more": true },
    "truncation": {
      "storage_capped": false,
      "response_capped": false,
      "total_before_max_urls": 4231,
      "total_detected_before_storage_cap": 4231,
      "discovery_capped": false,
      "discovery_cap_reason": null,
      "sitemaps_skipped": 0
    },
    "usage": { "credits": 1, "engine": "http", "proxy": "basic" }
  }
}
```

`links[]` carries `url` and nothing else — if you need a link's anchor text or its internal/external flag, that's the scrape response's `links` field, not this.

**ordering is stable across pages**, so paginating won't shuffle or drop entries under you. the most useful links come first.

`links[]` has no `source` field — don't expect one, and don't infer it from position.

**page 1 is the useful page** — if you only want a site's structure, one page at a small `limit` is usually enough.

### pagination

request `page: 1`, then keep going while `response_meta.pagination.has_more` is true:

```jsonc
"pagination": { "page": 1, "limit": 1000, "total": 4231, "total_pages": 5, "has_more": true }
```

`total` is the full count across all pages, not the length of this one.

### truncation

`truncation` tells you whether you're seeing everything the site has. it has two halves, and neither one is about `limit` or pagination — paging never loses urls, it just spreads the same stored list across pages.

**the stored list itself was cut shorter than what's really on the site:**

- **`response_capped`** — more links were eligible than `max_urls`, so the stored list was trimmed down to `max_urls`. discovery itself now *stops* at `max_urls`, so this is normally `true` only when home-page links pushed the total past a cap that sitemap discovery had already reached. it has nothing to do with `limit` or pagination.
- **`storage_capped`** / **`total_detected_before_storage_cap`** — the site had more urls than we retain in one map (the 100 000-url storage cap).
- **`total_before_max_urls`** — the eligible count before your `max_urls` cap applied. ⚠️ since discovery now *stops* at `max_urls`, this normally just equals `max_urls`. it is **not** "how many the site has".

**discovery gave up before it finished, so it never even found everything** — a map cut short here reports `response_capped: false`, since nothing was left over to trim:

- **`discovery_capped`** (boolean) — sitemap discovery stopped before reading every sitemap file it found. when true, the site has more pages than this map lists.
- **`discovery_cap_reason`** — which limit stopped it first, or `null` when nothing did: `max_urls` · `time` · `file_budget` · `depth` · `file_size` · `unread_files`.
- **`sitemaps_skipped`** (integer ≥ 0) — how many sitemap files were skipped or only partly read (too large, fetch failed, or a discovery limit hit).

#### reading `discovery_cap_reason`

branch on this field, because only one of its values is yours to fix:

| reason | what happened | what to do |
| --- | --- | --- |
| `null` | nothing capped discovery | the list is complete — stop |
| `max_urls` | your ceiling filled up while sitemap files were still unread | **retry with a higher `max_urls`** — the only reason where a bigger ceiling is guaranteed to help |
| `unread_files` | one or more sitemap files couldn't be read this time — a `429`/`5xx`, a `.xml.gz` (nothing here gunzips it), an html soft-404, json, or a truncated file | **a retry may help** — it's often a transient failure, and a map that hit this reason is cached for only 15 minutes (not the usual freshness window) so a retry soon is meant to work. but if it keeps reporting `unread_files`, the site is publishing something we can't read and the list won't grow — stop retrying |
| `time` · `file_budget` · `depth` · `file_size` | the site is bigger, deeper or slower than one discovery pass | raising `max_urls` won't help. narrow the target (map a subdomain, or `sitemap_only: true`), or accept a partial list |

⚠️ **a full page is not a signal, and neither is `response_capped`.** with the default `max_urls` of 5000, a large site returns exactly 5000 links with `response_capped: false` — because discovery stopped right on the cap, nothing was trimmed on the way out. that looks complete and isn't. `discovery_cap_reason: "max_urls"` is the honest signal, and a retry with a bigger `max_urls` is the fix.

## tips

- **check `discovery_cap_reason` before you trust the list.** it is the one field that distinguishes "this is the whole site" from "this is the first 5000 urls of it". only `max_urls` is worth a retry.
- **`sitemap_only: true` is faster and more predictable** when a site has a good sitemap and you don't care about homepage-only links.
- **filter to a section** with a path prefix before scraping — mapping a docs site and keeping only `/guide/` is the common shape.
- **map is cache-friendly.** repeat maps of the same root are served from cache within the freshness window; pass `cache.max_age: 0` when you specifically need a fresh inventory.
- **map costs credits** — a fresh map reports `engine: "http"`; a cached map reports `engine: "cache"`. those are the only map engine values. the returned `proxy` is the resolved `basic` or `advanced` tier, never `auto`. read `response_meta.usage.credits` for the result, and see <https://crawlbrulee.com/pricing>. don't assume it's free.

## see also

- scrape the urls you found: **crawlbrulee-scrape** · shared contract: **crawlbrulee-api**
- drive it from: **crawlbrulee-cli**, **crawlbrulee-mcp**, **crawlbrulee-sdk-js**, **crawlbrulee-sdk-python**
- docs: <https://crawlbrulee.com/docs/map>
