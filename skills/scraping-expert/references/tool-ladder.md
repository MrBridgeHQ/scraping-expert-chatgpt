# Tool Ladder - Choosing the Right Scraping Tool

The cost gap between scraping tools is enormous. `got-scraping` to `Camoufox` is roughly 100× in compute, memory, and proxy budget. Choosing one rung too high wastes money for years. Choosing one rung too low produces a scraper that fails constantly. This page is the decision tree.

The principle: **start one rung lower than you think you need, observe failure, escalate.** Failure is fast and cheap to observe. Over-engineering is expensive and invisible.

---

## The rungs (lowest to highest cost)

> Per-request **Cost** figures below are Apify Compute-Unit illustrations; the rung→resource shape (HTTP rungs are cheap and light, browser rungs are heavy) holds on any host.

### Rung 1 - `got-scraping` (raw HTTP)

**Cost**: ~$0.0001 per request. **Memory**: 128–256 MB. **Throughput**: hundreds of requests per second.

What it is: an HTTP client that mimics a real browser's TLS fingerprint, header order, and HTTP/2 settings. It is `node-fetch` with the JA3/JA4 problem solved. Built into Crawlee; can be used standalone.

Use when:
- You hit an internal JSON API (the most common case after diagnostic Phase 1).
- The target returns the data in initial HTML and has no JS challenge.
- You need maximum throughput and minimum cost.

Limits:
- Can't execute JavaScript. Sites with client-rendered content return empty shells.
- Cloudflare's JS challenges (`cf_chl_jschl_tk`) block this. Even though TLS is fine, no JS = no clearance.
- Some sites detect HTTP/2 settings frame anomalies even with `got-scraping`. Rare, but happens.

Implementation pattern (standalone):

```ts
import { gotScraping } from 'got-scraping';
const response = await gotScraping({
  url: 'https://www.example.com/api/products',
  proxyUrl: await proxyConfiguration.newUrl(),
  headerGeneratorOptions: {
    browsers: ['chrome', 'firefox'],
    operatingSystems: ['windows', 'macos'],
    locales: ['en-US'],
  },
  responseType: 'json',
});
```

Inside Crawlee, `BasicCrawler` + `gotScraping` covers this. Or use `HttpCrawler` if you want session pool integration without parsing.

### Rung 2 - `CheerioCrawler` (HTTP + DOM parsing)

**Cost**: ~$0.0002 per request. **Memory**: 256–512 MB. **Throughput**: 50–200 requests per second per worker.

What it is: `got-scraping` plus Cheerio for jQuery-style HTML parsing. The workhorse for static HTML scraping.

Use when:
- The data is in the initial HTML but extracting it requires DOM traversal.
- The site is static or server-rendered.
- Anti-bot is light (rate limiting, basic header checks).

Implementation pattern (Apify example; standalone Crawlee is identical minus `Actor.init/exit` - use `new ProxyConfiguration({ proxyUrls: [...] })` instead of `Actor.createProxyConfiguration`):

```ts
import { Actor } from 'apify';
import { CheerioCrawler, Dataset } from 'crawlee';

await Actor.init();
// datacenter proxies (on Apify: the `SHADER` group)
const proxyConfiguration = await Actor.createProxyConfiguration({ groups: ['SHADER'] });

const crawler = new CheerioCrawler({
  proxyConfiguration,
  useSessionPool: true,
  maxConcurrency: 10,
  maxRequestRetries: 3,
  async requestHandler({ $, request, log, enqueueLinks }) {
    const data = {
      url: request.loadedUrl,
      title: $('h1').first().text().trim(),
      price: $('[data-price]').attr('data-price') ?? $('.price-current').text().trim(),
    };
    await Dataset.pushData(data);
  },
});

await crawler.run([{ url: 'https://example.com/products' }]);
await Actor.exit('Completed');
```

Use this rung as the default for static HTML scraping. Most production scrapers belong here.

### Rung 3 - `PlaywrightCrawler` (real browser, default fingerprints)

**Cost**: ~$0.001–0.005 per request. **Memory**: 1–2 GB. **Throughput**: 5–20 requests per second per worker.

What it is: Crawlee's wrapper around Playwright (Chrome by default), with Crawlee's `BrowserPool` adding fingerprint generation and session management. The default browser tier.

Use when:
- The data is rendered client-side (SPA, React without SSR).
- The site requires JS execution to pass a basic challenge.
- Anti-bot is moderate (basic Cloudflare WAF, light Akamai/DataDome).

Implementation pattern:

```ts
import { PlaywrightCrawler } from 'crawlee';

const crawler = new PlaywrightCrawler({
  proxyConfiguration,
  useSessionPool: true,
  browserPoolOptions: {
    useFingerprints: true,
    fingerprintOptions: {
      fingerprintGeneratorOptions: {
        browsers: ['chrome', 'firefox'],
        devices: ['desktop'],
        operatingSystems: ['windows', 'macos'],
        locales: ['en-US'],
      },
    },
  },
  maxConcurrency: 5,
  navigationTimeoutSecs: 60,
  preNavigationHooks: [
    async ({ page }) => {
      // block heavy assets to save bandwidth
      await page.route('**/*.{png,jpg,gif,woff2,css}', route => route.abort());
    },
  ],
  async requestHandler({ page, request, log }) {
    await page.waitForSelector('.product-grid', { timeout: 30000 });
    const data = await page.evaluate(() => Array.from(document.querySelectorAll('.product')).map(el => ({
      name: el.querySelector('.name')?.textContent?.trim(),
      price: el.querySelector('.price')?.textContent?.trim(),
    })));
    for (const item of data) await Dataset.pushData(item);
  },
});
```

Crawlee 3.x ships built-in fingerprint injection, virtual session management, and `retryOnBlocked` - most stealth concerns are handled by the framework, not by manual code. Resist the urge to copy stealth recipes from old blog posts; they often conflict with Crawlee's defaults.

### Rung 3b - `AdaptivePlaywrightCrawler` (HTTP↔browser, automatic)

**Cost**: averages between Rung 2 and Rung 3 depending on content mix. **Memory**: same as Playwright.

What it is: a crawler that automatically detects whether each page needs JS rendering or can be parsed statically, and routes accordingly. Internally combines a static parser (Cheerio, BeautifulSoup, or Parsel) with PlaywrightCrawler.

Use when:
- The crawl spans multiple page types with mixed rendering needs.
- You want HTTP-only economics where possible without manual classification.
- The cost difference between rungs matters for the volume.

Implementation:

```ts
import { AdaptivePlaywrightCrawler } from 'crawlee';

const crawler = new AdaptivePlaywrightCrawler({
  proxyConfiguration,
  async requestHandler(ctx) {
    // ctx may be a Cheerio context OR a Playwright context
    // Use the unified helpers: ctx.querySelector, ctx.waitForSelector, ctx.parseWithCheerio
    const title = await ctx.querySelector('h1');
  },
});
```

The `RenderingTypePredictor` learns from observed pages - it gets smarter the longer the crawl runs. Excellent for crawling a whole site rather than known endpoints.

### Rung 4 - Camoufox (Firefox engine-level stealth)

**Cost**: ~$0.005–0.02 per request. **Memory**: 2–4 GB. **Throughput**: 1–5 requests per second per worker. **Per-page wall time**: 5–60 seconds.

What it is: a custom Firefox build with fingerprint spoofing patched at the C++ level. The only open-source tool in 2026 that consistently scores 0% on headless-detection tests. Exposes a Playwright-compatible API.

Use when:
- You hit Cloudflare aggressive (Bot Fight Mode), Akamai standard, DataDome standard, PerimeterX/HUMAN.
- The diagnostic showed challenge pages or behavioral analysis.
- Volume is moderate (the cost per request rules out commodity-scale crawling).

Implementation pattern (Apify example; the launcher/launchOptions block is identical on a standalone host - only the proxy wiring changes):

```ts
import { Actor } from 'apify';
import { PlaywrightCrawler } from 'crawlee';
import { firefox } from 'playwright';
import { launchOptions as camoufoxLaunchOptions } from 'camoufox-js';

await Actor.init();
// residential proxies (Apify: `RESIDENTIAL`)
const proxyConfiguration = await Actor.createProxyConfiguration({
  groups: ['RESIDENTIAL'],
  countryCode: 'US',
});

const crawler = new PlaywrightCrawler({
  proxyConfiguration,
  launchContext: {
    launcher: firefox,
    launchOptions: await camoufoxLaunchOptions({
      headless: true,
      proxy: await proxyConfiguration.newUrl(),
      geoip: true,    // align timezone, locale, screen with proxy IP
      humanize: true, // built-in mouse/scroll humanization
    }),
  },
  // disable Crawlee's default fingerprints - Camoufox handles this at engine level
  browserPoolOptions: { useFingerprints: false },
  maxConcurrency: 2,
  navigationTimeoutSecs: 120,
  requestHandlerTimeoutSecs: 300,
  // ... rest as Rung 3
});
```

Watch-outs:
- Camoufox went through a maintenance gap in 2025 and is back under active development. Pin the version explicitly.
- The base Firefox version drives some fingerprint inconsistencies. Test on the target before committing to a long campaign.
- Camoufox is Firefox-based; sites that test for Spidermonkey-specific behavior (rare, but Interstitial-class WAFs do this) will detect it.
- Resource consumption is real. Plan for 2–4 GB RAM per browser instance, low concurrency.

### Rung 4b - Other anti-detect browsers and approaches

Camoufox is the open-source default, but it is not the only option in this rung. Adjacent tools serve specific niches:

- **Patchright** (a Playwright fork that fixes CDP leaks via JavaScript isolation) - easier integration than Camoufox if you're already on Chromium, but only ~67% headless-detection reduction vs. Camoufox's near-100%. Useful for moderate targets where you'd rather not switch to Firefox.
- **noDriver** (drives Chrome via raw CDP without WebDriver) - strips out Playwright entirely, avoiding the `Runtime.enable` CDP call that detection systems flag. Lighter on resources than full Playwright, but limited to basic stealth - won't pass the hardest targets.
- **Octo Browser, GoLogin, Multilogin** - commercial anti-detect browsers with profile management. Each "profile" is a complete fingerprint set (UA, fonts, canvas, screen, plugins) maintained as a stable identity across sessions. Useful for ongoing scraping where the same identity must persist for weeks. Generally not used in headless cloud deployments - they're desktop tools - but the concept (stable identity per "user") is the same as Crawlee's session pool with persistent fingerprints.
- **Kameleo** - similar tier to Octo, with Selenium and Playwright bridges.

For any headless cloud deployment (Apify Actors included), Camoufox is the practical choice. The commercial anti-detect browsers don't fit the headless cloud-deployment model. Patchright is worth knowing as a lighter alternative when full Camoufox is overkill.

### Rung 5 - Managed scraping APIs

**Cost**: $0.001–0.05 per request depending on tier. **Memory**: same as a thin HTTP client (the heavy lifting happens on the provider's side). **Throughput**: provider-dependent.

What it is: a paid third-party that handles proxies, browsers, fingerprints, CAPTCHA, and unblocking, and returns the rendered page. You make a normal HTTP call to their endpoint, get HTML or extracted data back.

Vendors and use cases:
- **Firecrawl** - best for AI/LLM extraction pipelines. Markdown output, schema-based extraction. Strong default behavior.
- **ZenRows** - generalist, good Cloudflare/DataDome bypass. Per-call pricing.
- **ScrapFly** - generalist, strong residential pool. Per-call.
- **ScraperAPI** - generalist, simpler API surface. Generous free tier.
- **Bright Data Scraping Browser** - premium, managed Chromium with built-in unblocking. Connect via CDP.

Use when:
- The target is hard (LinkedIn, Amazon, ticketing, sneakers, banking).
- DIY economics break down (ROI of building and maintaining a Camoufox-grade scraper exceeds per-call cost of a managed API).
- Speed-to-market matters more than cost-per-call.
- You already have credits on Firecrawl and prefer to use them up.

Implementation: standard HTTP client (`got-scraping`, `axios`, `fetch`) calling the provider's endpoint with the target URL as a parameter. See `managed-apis.md` for vendor-specific details.

---

## Adjacent: Stagehand (AI-element-selection layer on Playwright)

Stagehand is an AI-driven browser automation framework that sits **atop Playwright** rather than replacing it. Instead of CSS selectors, the developer (or end user) describes the target element in natural language; an LLM resolves it to a Playwright action. It appears in production on several public Apify Store Actors (e.g. `apify/ai-web-scraper` at ~$25/1,000 page extractions with absorbed LLM cost, and community-maintained Stagehand crawlers that take a user-supplied LLM API key).

The canonical Crawlee + Stagehand integration is published as the npm package **`@crawlee/stagehand`**. Use this package when wrapping Stagehand inside a Crawlee request queue / autoscaling pool.

**Where it sits on the ladder:** Stagehand is **not a separate rung** - it's a layer on top of Rung 3 (Playwright) or 4 (Camoufox via Patchright). Cost = Playwright cost + per-page LLM call cost ($0.01–0.05 typical). The economics only work for use cases where:

- The target site changes frequently (selectors break weekly) → Stagehand's AI element resolution saves selector-maintenance cost.
- The end user supplies a one-shot extraction prompt instead of you writing extractor code → Stagehand turns a generic browser into a prompt-driven scraper. This is the business model behind generic AI-web-scraper Actors.
- Single LLM call per page (extraction, not multi-step orchestration) - this is the boundary between a scraper-with-AI and a full orchestration agent.

### Stagehand agent modes - configurable cost-vs-tolerance axis

Stagehand exposes three agent modes that trade cost for visual-page tolerance. Exposing the mode as a user-facing input choice is the recommended pattern for any Stagehand-based Actor:

| Mode | What it does | When to use | Cost |
|---|---|---|---|
| `dom` | Reads page DOM to determine actions | Stable static HTML, predictable structure | Lowest LLM token usage |
| `hybrid` | DOM + visual understanding (requires vision-capable model: Gemini, Claude) | Complex / dynamic layouts where DOM alone is insufficient | Medium token usage |
| `cua` | Computer-use agent - operates purely from screenshots (Anthropic Computer Use / OpenAI `computer-use-preview`) | Heavy JS interaction, drag-and-drop, canvas-based UIs | Highest token usage |

**Recommendation:** expose the mode as a user-facing input toggle (default `dom`, allow opt-in to `hybrid`/`cua` for difficult targets) so users can opt in to higher cost only when their target requires it.

**When NOT to use Stagehand:** stable target sites with predictable structure (the per-page LLM cost is wasted), commodity-volume scrapes (LLM cost dominates the budget), or selectors that don't actually drift (Stagehand's value prop disappears).

**Integration:** Stagehand is Crawlee-compatible via `@crawlee/stagehand` (the canonical npm package). See also [crawlee.dev/js/docs/guides/stagehand-crawler-guide](https://crawlee.dev/js/docs/guides/stagehand-crawler-guide). Drop-in for existing Playwright scripts with `stagehand.observe()` / `stagehand.act()` / `stagehand.extract()` calls replacing `page.locator()` chains.

**Pricing implication:** Stagehand-based Actors have a few common LLM-cost-handling options:

- **Absorbed cost:** the Actor's PPE unit price covers the per-page LLM token cost + the developer's margin. Best for a self-serve / non-technical audience.
- **Bring-your-own-key (BYO-key):** the end user provides their own LLM provider `apiKey` in the input; the Actor charges only the orchestration overhead. Best for a developer audience that already holds OpenAI / Anthropic / Google AI Studio keys.
- **Sub-Actor pass-through with transparent disclosure:** name and link the sub-Actors so the cost breakdown is visible. Best when the orchestration variance is high.

The choice depends on target audience and margin tolerance.

---

## Adjacent: Domain-aware routing pattern (multi-source content extraction)

For Actors orchestrating extraction from **multiple high-value sources** (news media, financial publishers, e-commerce platforms), an alternative to the single-generic-extractor pattern is **domain-aware routing**: route per-target-domain to a dedicated specialist scraper, falling back to a generic extractor only when the domain isn't in the routing table.

**Seen in production on `kay_abraham/news-intelligence-agent`** - routes 7 Tier-1 news publishers each to a dedicated `xtracto/*-scraper`:

```
nytimes.com       → xtracto/nytimes-scraper
bloomberg.com     → xtracto/bloomberg-news-article-scraper
reuters.com       → xtracto/reuters-scraper
ft.com            → xtracto/ft-scraper
theguardian.com   → xtracto/guardian-scraper
aljazeera.com     → piotrv1001/aljazeera-scraper
bbc.com / bbc.co.uk → nexgendata/bbc-news-scraper
<everything else> → lukaskrivka/article-extractor-smart
                    → apify/website-content-crawler (last resort)
```

### When to use domain-aware routing

- **Anti-bot defenses are domain-specific.** Tier-1 publishers (paywalls + cookie walls + bot detection) require domain-tuned scrapers; a generic Cheerio + Playwright extractor gets blocked. Each specialist scraper has its own bot-evasion logic tuned to that domain.
- **Multi-source agents that need consistency across sources.** Different paywalls + cookie banners + DOM structures across 5-20 source domains. A single generic extractor produces inconsistent results; per-domain specialists produce uniform output.
- **High-value content where extraction quality matters more than orchestration simplicity.** Worth the routing-table maintenance cost.

### When NOT to use domain-aware routing

- **Single-source agents.** Don't add domain-aware routing for "one domain plus a few similar ones" - just use the right specialist (or a generic extractor) directly.
- **Low-stakes content** where partial-extraction failures are tolerable. The maintenance cost of the routing table outweighs the quality gain.
- **High-volume commodity scrapes.** The per-call cost of dedicated specialists adds up faster than generic-extractor cost.

### Architectural implications

- **The routing logic is the agent's core IP.** Maintaining the routing table (adding new domain specialists, removing deprecated ones, tuning fallback thresholds) is the developer's ongoing work.
- **Pricing transparency.** Domain-aware routing pairs naturally with transparent sub-Actor disclosure - the routing table itself becomes the disclosure surface. List each domain + its dedicated specialist + the fallback chain in a `## Tech stack under the hood` section of the README.
- **Retry strategy.** If a dedicated specialist returns empty, automatically retry through the generic fallback. Document this in the README so users understand the coverage guarantees.

### Implementation note

Use `Promise.allSettled()` across the per-domain calls so partial failures don't abort the entire run. Surface the per-domain success/failure rates in the run log so users can see which specialists succeeded for their input batch.

---

## The decision matrix

A more compact view, indexed by what the diagnostic told you:

| Diagnostic signal | First choice | Fallback |
|---|---|---|
| Internal API, no auth | `got-scraping` | - |
| Internal API + cookie auth | `got-scraping` + auth refresh via Playwright | - |
| Static HTML, no anti-bot | `CheerioCrawler` + datacenter | + sessions |
| Static HTML + naive rate limit | `CheerioCrawler` + datacenter + sessions + throttling | residential |
| Static HTML + Cloudflare WAF default | `CheerioCrawler` + residential | `PlaywrightCrawler` |
| JS-rendered, no anti-bot | `PlaywrightCrawler` defaults | - |
| JS-rendered + Cloudflare Bot Fight | `PlaywrightCrawler` + residential + fingerprints | Camoufox |
| Akamai, DataDome, PerimeterX | Camoufox + residential | Managed API |
| LinkedIn, Amazon, hard targets | Managed API | - |
| Search results across many sites | `AdaptivePlaywrightCrawler` | per-site decision |
| Google SERP | SERP proxy (Apify: `GOOGLE_SERP`) + `CheerioCrawler` | SerpApi (managed) |

---

## Anti-patterns

These show up regularly. Resist them.

### "Let's just use Playwright for everything, it's safer"

False. Playwright is 10× more expensive than Cheerio per request and adds 5–30 seconds of startup time. For a static HTML site, it gains you nothing and costs you everything. The discipline of starting at the right rung pays off across every scraper you ever ship.

### "I'll add stealth plugins to Playwright instead of upgrading to Camoufox"

`playwright-extra` and `puppeteer-extra-plugin-stealth` have not seen meaningful updates since 2023. The patches were written for older detection vectors; modern WAFs see through them. If you actually need anti-detection, jump straight to Camoufox or a managed API. The plugin stack is a 2022 solution.

### "Crawlee's default fingerprints conflict with Camoufox"

True. Disable Crawlee's `useFingerprints` when running Camoufox - Camoufox handles fingerprint at the engine level and double-spoofing creates inconsistencies that flag the request.

### "I'll write my own User-Agent rotation"

Don't. `got-scraping` and Crawlee's fingerprint generator handle this with internally consistent sets (UA + Accept-Language + sec-ch-ua + viewport + timezone all matching). A hand-rolled UA list with mismatched headers is detected immediately.

### "I'll skip the sessions, simpler"

Sessions are the difference between a scraper that runs for an hour and one that runs for a week. Each session represents one "user" with consistent cookies, IP (when paired with sticky proxies), and fingerprint. Burning a session is normal; the pool maintains health. Without sessions, you burn the IP itself.

### "Headless: false will fix anti-bot"

Sometimes, but rarely the right answer. Modern anti-bot doesn't rely on the headless flag alone - it cross-checks dozens of signals. If `headless: false` "fixes" the issue, it usually means the bot was running with Crawlee's defaults stripped out. Restore the defaults; that's likely what fixed it, not the visible browser.

### "Let's add a CAPTCHA solver"

The presence of a CAPTCHA usually means you've already failed earlier checks. Solving the CAPTCHA gets you one request; the next one will get challenged again. The right move is to upgrade the upstream config (proxy, fingerprint, session) so CAPTCHAs don't appear in the first place. CAPTCHA solving services are a last resort, not a default.

---

## Apify infrastructure considerations (Apify deployment only)

> Apify-specific. On a standalone host the rung→resource mapping is the same idea (HTTP rungs need little RAM; browser rungs need 2-4 GB), but the base-image / memory-tier / Standby columns below are Apify config (a separate build concern).

Per-rung Actor configuration:

| Rung | Memory (MB) | Build base image | `usesStandbyMode` |
|---|---|---|---|
| got-scraping standalone | 256 | apify/actor-node:20 | false |
| CheerioCrawler | 512–1024 | apify/actor-node:20 | false |
| PlaywrightCrawler | 2048–4096 | apify/actor-node-playwright-chrome:20 | false |
| AdaptivePlaywrightCrawler | 2048–4096 | apify/actor-node-playwright:20 | false |
| Camoufox | 2048–4096 | apify/actor-node-playwright-firefox:20 (custom layer for Camoufox) | false |
| MCP Standby server | 256–512 (per request) | apify/actor-node:20 | true |

Standby mode is for MCP servers exclusively (see `apify-patterns.md`). All other scrapers use the default run mode.

---

## Cost modeling cheat sheet

Before committing to a rung, calculate per-call cost roughly. The formula below is in Apify Compute Units; on any other host substitute your own compute price (per-GB-hour or per-vCPU-second) - the rung→resource shape that drives the cost is the same everywhere:

```
Per-call cost ≈
    (memory_MB / 4096) × $0.40/CU/hour × (wall_seconds / 3600)
  + proxy_credits_per_call
```

(Rough Apify CU cost; adjust for current pricing.)

Examples at moderate proxy use:
- got-scraping at 256 MB, 0.5 s, datacenter proxy: ~$0.0002 per call
- CheerioCrawler at 512 MB, 1 s, datacenter: ~$0.0004 per call
- PlaywrightCrawler at 2 GB, 5 s, datacenter: ~$0.003 per call
- Camoufox at 4 GB, 30 s, residential: ~$0.04 per call
- Managed API: $0.001–0.05 per call (provider-dependent)

What matters on any host is cost-per-result vs value-per-result. If monetizing on Apify, the PPE price must absorb the cost-per-result with margin. A scraper at $0.04 per call sold at $0.05 PPE gives 25% margin, which is too thin given retries and free-plan absorption. Aim for 3–5× the underlying cost.
