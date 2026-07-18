# Anti-Bot Blocking — Solutions Reference

Researched 2026-07-17 across hermes-agent.nousresearch.com, searxng docs,
agent-browser.dev, scrapling docs, tilion.dev, and GitHub community.

## The Problem

Since July 2025, Cloudflare blocks AI agents by default. The block is silent:
the page returns 200 with a "Just a moment..." HTML body. An LLM reads the
block page as if it were real content and produces confidently wrong answers.

12 commonly-blocked sites: Indeed, Glassdoor, ZipRecruiter, Crunchbase,
Product Hunt, Capterra, Upwork, Fiverr, StockX, DoorDash, Coinbase, Udemy.

## Hermes' Built-in Browser

Hermes uses `agent-browser` (Vercel Labs, Rust binary) + Chrome 149 (Chrome
for Testing) via CDP. Location: `/opt/hermes/node_modules/agent-browser/`,
Chrome at `/opt/data/.agent-browser/browsers/chrome-149.0.7827.54/`.

Capabilities: navigate, click, type, snapshot (accessibility tree), scroll,
console eval, screenshot. No stealth features, no fingerprint spoofing.

## Solution Tier List

### Tier 0: Detect + Skip (free, always available)
Check page title after `browser_navigate`. If "Just a moment..." → blocked.
Skip URL, move to next search result. Do NOT read the block page as content.

### Tier 1: Scrapling Stealth (free, installed)
```bash
PLAYWRIGHT_BROWSERS_PATH=/opt/data/.playwright \
  /opt/hermes/.venv/bin/scrapling extract stealthy-fetch '<URL>' output.md \
  --solve-cloudflare --block-webrtc --hide-canvas
```
Playwright Chromium 149 at `/opt/data/.playwright/chromium-1228/`.
Pass rate: ~5/8 Cloudflare-protected sites. Adds 5-15s per page.

### Tier 2: Camofox (free, not installed — rejected)
Firefox fork with C++ fingerprint spoofing. Docker-based, separate container.
Pass rate: ~5/8. Rejected for our setup: adds complexity for infrequent use,
and Scrapling achieves the same with zero infrastructure cost.

### Tier 3: Browserbase / Tool Gateway (paid)
Cloud browsers with residential proxies, CAPTCHA solving, random fingerprints.
Pass rate: 8/8. Available via Nous Portal subscription (`hermes setup --portal`)
or Browserbase API key (`BROWSERBASE_API_KEY` in .env).

## Benchmark Data (from tilion.dev, July 2026)

Site          | Anti-bot      | Naive     | Stealth   | Cloud
--------------|---------------|-----------|-----------|------
Indeed        | Cloudflare    | blocked   | pass      | pass
Zillow        | PerimeterX    | blocked   | pass      | pass
Walmart       | Akamai        | blocked   | pass      | pass
StockX        | Cloudflare    | blocked   | pass      | pass
Booking       | proprietary   | blocked   | pass      | pass
Amazon        | CloudFront    | blocked   | blocked   | pass
Etsy          | DataDome      | blocked   | blocked   | pass
G2            | DataDome      | blocked   | blocked   | pass

Naive = curl with desktop Chrome UA. Stealth = open-source stealth browser.
Cloud = Tilion Cloud (hosted stealth browser).
