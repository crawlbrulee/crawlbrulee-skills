# 🍮 crawlbrulee skills

[![agent skills](https://img.shields.io/badge/agent%20skills-11-blue?style=flat-square)](https://agentskills.io/)
[![license](https://img.shields.io/badge/license-Apache--2.0-blue?style=flat-square)](./LICENSE)

**EU-native web scraping for AI agents & developers.**

official [agent skills](https://agentskills.io/) for [crawlbrulee](https://crawlbrulee.com). one call turns any url into clean markdown, screenshots, metadata and links, and maps a site's urls. the fetch, the render, the cache and your result stay on EU servers.

drop these into any skills-aware coding agent (Claude Code, Cursor, Codex, Gemini, …) and it will know how to drive crawlbrulee through whichever interface fits — the cli, the mcp server, the js/ts or python sdk, or raw http.

**get a free api key** → [dashboard.crawlbrulee.com](https://dashboard.crawlbrulee.com)

## skills

**start here** — the entry point routes an agent to the rest:

| skill | use it to… |
| --- | --- |
| **crawlbrulee** | start here — what crawlbrulee does, set up the api key, and pick an interface. |

**what the api does** — the contract, whichever interface you call from:

| skill | use it to… |
| --- | --- |
| **crawlbrulee-api** | the shared contract: auth, endpoints, the usage object, caching, proxies, errors — plus raw http. |
| **crawlbrulee-scrape** | turn a url into markdown, html, links, images, metadata, or a pdf. |
| **crawlbrulee-screenshots** | capture viewport / full-page / mobile shots, light or dark, and slice tall pages. |
| **crawlbrulee-scrape-async** | run scrapes as background jobs, or get a signed webhook on completion. |
| **crawlbrulee-map** | discover which urls exist on a site before scraping them, one sitemap at a time if you like, or only what changed since a date. |

**how to call it** — pick the one that fits where your work happens:

| skill | use it to… |
| --- | --- |
| **crawlbrulee-cli** | scrape, map, and check usage from the terminal (`npx crawlbrulee`). |
| **crawlbrulee-mcp** | give an AI agent crawlbrulee as native mcp tools, through the hosted server and a browser sign-in. |
| **crawlbrulee-mcp-local** | the same tools from `@crawlbrulee/mcp` on your machine, with an api key, plus your own headers and cookies, webhooks and raw html. |
| **crawlbrulee-sdk-js** | call crawlbrulee from Node.js / TypeScript / Deno / Bun (`@crawlbrulee/sdk`). |
| **crawlbrulee-sdk-python** | call crawlbrulee from Python, sync or async (`crawlbrulee` on PyPI). |

each skill lives in `skills/<name>/SKILL.md` in the open Agent Skills format (`name` + `description` frontmatter).

## just the values you need

when an agent needs a few values from a page — prices, titles, links, a table's rows — the skills teach it to ask for them by css selector with `extract.elements` instead of reading the whole page. it costs no extra credits:

```json
{
  "url": "https://shop.example.com/books/",
  "extract": {
    "markdown": false,
    "metadata": false,
    "elements": {
      "heading": "h1",
      "books": { "selector": "article.book", "all": true, "fields": { "price": ".price" } }
    }
  }
}
```

the result carries `"elements": { "heading": "Books", "books": [{ "price": "€18.90" }, …] }`. selector rules and limits: [targeted extraction](https://crawlbrulee.com/docs/scrape/targeted-extraction).

## install

install with the [skills](https://www.skills.sh/) cli:

```bash
# all skills
npx skills add crawlbrulee/crawlbrulee-skills

# a single skill
npx skills add crawlbrulee/crawlbrulee-skills@crawlbrulee-cli
```

you can also copy any `skills/<name>/` folder into your agent's skills directory by hand.

## install as a plugin

this repo is also a plugin. a plugin install gives you the same skills **plus** a connection to the hosted crawlbrulee mcp server, `https://api.crawlbrulee.com/mcp`. nothing runs on your machine. these commands work straight from the repo, no store listing needed.

```bash
# Claude Code (and Claude Cowork)
claude plugin marketplace add crawlbrulee/crawlbrulee-skills
claude plugin install crawlbrulee@crawlbrulee

# Codex
codex plugin marketplace add crawlbrulee/crawlbrulee-skills
codex plugin add crawlbrulee@crawlbrulee

# Gemini CLI
gemini extensions install https://github.com/crawlbrulee/crawlbrulee-skills

# Antigravity, Cursor, and any other skills-aware agent
npx skills add crawlbrulee/crawlbrulee-skills --agent antigravity
npx skills add crawlbrulee/crawlbrulee-skills --agent cursor
```

Grok Build reads Claude Code marketplaces and plugins as they are, so the `claude plugin marketplace add` line above is also how you add it to Grok.

**sign in, no api key.** the first time the agent uses the server, your host opens a crawlbrulee sign-in page in your browser. you sign in, pick the organization to connect and approve. the connection gets its own key, which you can see and delete with your other api tokens in the dashboard; deleting it disconnects the app. in Claude Code, run `/mcp` and pick `crawlbrulee` to sign in.

prefer the local server, with an api key from your environment? add `@crawlbrulee/mcp` yourself, as in [the mcp docs](https://crawlbrulee.com/docs/mcp) and the `crawlbrulee-mcp-local` skill. the cli, the sdks and the curl examples in the skills still read `CRAWLBRULEE_API_KEY`.

## what gets sent, and where

- the mcp server connection, and the curl, cli and sdk examples in the skills, send your requests (the url and the options you pick) with your key to the crawlbrulee api, `https://api.crawlbrulee.com`.
- the sign-in for the mcp server runs on `https://dashboard.crawlbrulee.com`.
- screenshots and pdfs come back as signed links on `cdn.crawlbrulee.com` that expire 24 hours after the scrape.
- the api runs on EU servers. we keep scrape, map and screenshot results only so caching works, and delete them no later than 90 days after they're created.

[privacy policy](https://crawlbrulee.com/legal/privacy-policy) · [data processing agreement](https://crawlbrulee.com/legal/dpa)

## part of the crawlbrulee toolkit

one api, many ways to call it:

- **[js/ts sdk](https://github.com/crawlbrulee/crawlbrulee-js)** — `@crawlbrulee/sdk`
- **[python sdk](https://github.com/crawlbrulee/crawlbrulee-py)** — `crawlbrulee` on PyPI
- **[cli](https://github.com/crawlbrulee/crawlbrulee-cli)** — `npx crawlbrulee`
- **[mcp server](https://github.com/crawlbrulee/crawlbrulee-mcp)** — `@crawlbrulee/mcp`, for ai agents
- **agent skills** — this bundle

docs: [crawlbrulee.com/docs](https://crawlbrulee.com/docs) · dashboard: [dashboard.crawlbrulee.com](https://dashboard.crawlbrulee.com) · pricing: [crawlbrulee.com/pricing](https://crawlbrulee.com/pricing)

## license

[Apache-2.0](./LICENSE). see [NOTICE](./NOTICE).
