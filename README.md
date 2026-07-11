# scraping-expert - ChatGPT / Codex Agent Skill

Senior-level doctrine for building web scrapers that run for months, not selectors that break tomorrow. An Agent Skill **for ChatGPT / Codex**. Platform-agnostic: standalone scripts, cron jobs, serverless, web-app backends, or Apify Actors.

Scraping is engineering, not selectors. The gap between a script that works once and a scraper that survives in production is a sequence of decisions made before a single line of code.

## What it does

- **Diagnose first.** Identify the anti-bot vendor and the JS-rendering needs, and run the hidden-JSON-API check before reaching for a browser.
- **Pick the right tool on a cost ladder:** got-scraping, Cheerio, Playwright, Puppeteer, Camoufox, and managed APIs (Firecrawl, ZenRows, ScrapFly, ScraperAPI, Bright Data).
- **Beat anti-bot economically** (Cloudflare, DataDome, PerimeterX, Akamai, AWS WAF): proxy strategy, browser and TLS fingerprinting, session pools, rate control.
- **Debug 403 and 429 and selector rot,** and build in resilience, caching, and graceful failure handling so the scraper keeps running.

## Core insight

Nine times out of ten, an internal JSON API exists and you are about to scrape HTML for nothing. And anti-bot is an economic cost function, not an absolute wall: keep the cost-per-result below the value-per-result, under the trust threshold the target has set. Where a target uses government-ID verification, that is a hard limit, not a cost function, and this skill does not engineer around it.

## When to use

Building, modifying, or debugging any scraper; anti-bot bypass; proxy or fingerprint strategy; choosing between scraping tools or managed APIs; 403 or 429 debugging; caching; selector breakage; as soon as a target URL or site name appears in a scraping context.

## Installation

### ChatGPT / Codex, project level

```bash
cp -r skills/scraping-expert /path/to/your-project/.codex/skills/
```

### ChatGPT / Codex, global level

```bash
cp -r skills/scraping-expert ~/.codex/skills/
```

## Use it

Describe the scraping problem in natural language:

- "This site returns 403 with plain requests, how should I scrape it?"
- "Is there a hidden API behind this page before I reach for Playwright?"
- "My scraper worked yesterday and now gets 429s, help me diagnose."
- "Choose a tool and a proxy strategy to scrape this site at scale."
- "Review my scraper for resilience and failure handling."

## What is inside

The skill lives in [`skills/scraping-expert/`](skills/scraping-expert/): a `SKILL.md` plus a `references/` library (diagnostic, tool-ladder, anti-bot-strategies, hidden-apis, selector-strategy, error-handling-ppe, testing-fixtures, managed-apis, apify-patterns, legal-ethics).

It is the doctrine and architecture layer (what to build and why): it pairs with the tools that scaffold, audit, or wrap a scraper. Ethical and legal boundaries live in `skills/scraping-expert/references/legal-ethics.md` (no bypassing government-ID verification, respect for terms and law, no scraping from your local machine).

## License

See `LICENSE`.

---

Part of the **[mr-bridge.com](https://mr-bridge.com)** toolkit for scraping, data, and content automation:
[Scrapers](https://mr-bridge.com/scrapers) · [MCP servers](https://mr-bridge.com/mcp-servers) · [AI workflows](https://mr-bridge.com/ai-workflows) · [Studies](https://mr-bridge.com/studies) · [Articles](https://mr-bridge.com/articles) · [Solutions](https://mr-bridge.com/solutions)

---

*Part of the [MrBridge Agent Skills catalog](https://github.com/MrBridgeHQ/skills). Browse them all at [mr-bridge.com/skills](https://mr-bridge.com/skills).*
