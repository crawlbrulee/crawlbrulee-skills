# 🍮 crawlbrulee skills

[![agent skills](https://img.shields.io/badge/agent%20skills-10-blue?style=flat-square)](https://agentskills.io/)
[![license](https://img.shields.io/badge/license-Apache--2.0-blue?style=flat-square)](./LICENSE)

official [agent skills](https://agentskills.io/) for [crawlbrulee](https://crawlbrulee.com) — the web-scraping api that turns any url into clean markdown, html, links, images, screenshots, and metadata, and maps a site's urls.

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
| **crawlbrulee-scrape** | turn a url into markdown, html, links, images, or metadata. |
| **crawlbrulee-screenshots** | capture viewport / full-page / mobile shots, and slice tall pages. |
| **crawlbrulee-scrape-async** | run scrapes as background jobs, or get a signed webhook on completion. |
| **crawlbrulee-map** | discover which urls exist on a site before scraping them. |

**how to call it** — pick the one that fits where your work happens:

| skill | use it to… |
| --- | --- |
| **crawlbrulee-cli** | scrape, map, and check usage from the terminal (`npx crawlbrulee`). |
| **crawlbrulee-mcp** | expose crawlbrulee as native mcp tools to an ai agent (`@crawlbrulee/mcp`). |
| **crawlbrulee-sdk-js** | call crawlbrulee from Node.js / TypeScript / Deno / Bun (`@crawlbrulee/sdk`). |
| **crawlbrulee-sdk-python** | call crawlbrulee from Python, sync or async (`crawlbrulee` on PyPI). |

each skill lives in `skills/<name>/SKILL.md` in the open Agent Skills format (`name` + `description` frontmatter).

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

this repo is also a plugin. a plugin install gives you the same ten skills **plus** the crawlbrulee mcp server, wired to start with `npx -y @crawlbrulee/mcp`. these commands work straight from the repo — no store listing needed.

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

**the api key.** Claude Code, Cursor, and Gemini CLI ask you for it at install time and keep it in your system keychain. every other host reads it from the `CRAWLBRULEE_API_KEY` environment variable, so export it before you start the agent:

```bash
export CRAWLBRULEE_API_KEY=cwbl_...
```

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
