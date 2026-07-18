# Hermes Search Survival Guide

[English](README.md) | [中文](README.zh.md)

A complete, battle-tested search configuration for [Hermes Agent](https://hermes-agent.nousresearch.com).
Stop debugging search engines that will never work. Start with the two that do.

---

## The Engine Mortality Table

After weeks of testing on a self-hosted SearXNG instance:

| Engine | Status | Results | Verdict |
|--------|:------:|:-------:|--------|
| **bing** | ✅ | ~10/query | **Use it.** Needs `timeout: 15.0`. |
| **mojeek** | ✅ | ~9-10/query | **Use it.** Independent index, rarely blocks. |
| brave | ❌ | 0 | Dead. Permanently banned on self-hosted. |
| duckduckgo | ❌ | 0 | Dead. CAPTCHA on all self-hosted. |
| startpage | ❌ | 0 | Dead. CAPTCHA on all self-hosted. |
| presearch | ❌ | 0 | Dead. Access denied. |
| qwant | ❌ | 0 | Dead. Access denied. |
| google | 🔒 | 0 | Requires paid Google Custom Search API key. |

**The takeaway**: 7 engines, 2 survivors. Every self-hosted SearXNG user rediscovers this
the hard way. This repo is the shortcut.

---

## What's in This Repo

```
hermes-search-survival-guide/
├── README.md                              ← You are here
├── README.zh.md                           ← 中文版
├── AGENTS.md                              ← Agent auto-loads this
├── docker-compose.yml                     ← Hermes + SearXNG + Valkey
├── config/
│   └── config.yaml.template               ← Hermes web search config
├── docker/
│   └── searxng/
│       └── settings.yml                   ← Full 2782-line config, working
├── skills/
│   ├── hermes-web-search-debugging/       ← Diagnose any search failure
│   │   ├── SKILL.md
│   │   └── references/
│   ├── web-search-strategy/               ← Multi-round search methodology
│   │   ├── SKILL.md
│   │   └── references/
│   └── searxng-config/                    ← SearXNG setup (standalone)
│       ├── SKILL.md
│       └── templates/
│           └── settings.yml               ← Minimal config: only the 2 working engines
├── .gitignore
└── LICENSE
```

---

## Quick Start

### Prerequisites

- Docker + Docker Compose
- [Hermes Agent](https://hermes-agent.nousresearch.com) installed (`~/.hermes/`)
- Port 8085 free (for SearXNG)

### 1. Clone

```bash
git clone https://github.com/wu1chenghui/hermes-search-survival-guide.git
cd hermes-search-survival-guide
```

### 2. Add web search config to ~/.hermes

**If you already have `~/.hermes/config.yaml`** (most Hermes users do), add these
lines under the existing content:

```yaml
web:
  search_backend: "searxng"
  extract_backend: "firecrawl"
```

**If starting fresh**, copy the template:

```bash
cp config/config.yaml.template ~/.hermes/config.yaml
```

Then copy the skills:

```bash
mkdir -p ~/.hermes/skills
cp -r skills/hermes-web-search-debugging ~/.hermes/skills/
cp -r skills/web-search-strategy ~/.hermes/skills/
cp -r skills/searxng-config ~/.hermes/skills/
```

### 3. Copy SearXNG settings

```bash
mkdir -p docker-data/searxng/config docker-data/searxng/cache
cp docker/searxng/settings.yml docker-data/searxng/config/
```

For a minimal config (only bing + mojeek + non-web engines), use:
```bash
cp skills/searxng-config/templates/settings.yml docker-data/searxng/config/
```

### 4. Start

```bash
docker compose up -d
```

### 5. Install ddgs (optional, for the second backend)

```bash
docker compose exec hermes bash -c "uv pip install --target /opt/data/pip-packages ddgs h2 httpcore brotli fake-useragent lxml primp"
docker compose exec hermes bash -c "uv pip install --target /opt/data/pip-packages 'scrapling[all]'"
```

### 6. Verify

```bash
curl -s --max-time 30 "http://localhost:8085/search?q=test&format=json" | python3 -c "
import sys,json
d=json.load(sys.stdin)
print(f'Results: {len(d.get(\"results\",[]))} | Unresponsive: {d.get(\"unresponsive_engines\",[])}')
# Expected: 17-20 results, unresponsive list empty or presearch only
"
# If you get 0 results or an error, see "Known Pitfalls" below.
```

Your Hermes container already has `SEARXNG_URL` and `search_backend: searxng` configured.
It is now using SearXNG for all web_search calls. No further setup needed.

---

## Skills Included

| Skill | What it does |
|-------|-------------|
| `hermes-web-search-debugging` | Diagnose any web_search failure — 8 backends, silent fallback detection, engine health checks |
| `web-search-strategy` | 4-phase research methodology, Cloudflare bypass via Scrapling, search budget control |
| `searxng-config` | Standalone SearXNG configuration guide — engine reliability table, minimal config, diagnostics |

---

## Architecture

```
┌─────────────┐     ┌───────────────┐     ┌──────────┐
│   Hermes    │────▶│  searxng-core │────▶│   bing   │
│  (Docker)   │     │   (Docker)    │     │  mojeek  │
│             │     │    :8080      │     │  arxiv   │
│             │     └───────┬───────┘     │   ...    │
└──────┬──────┘             │             └──────────┘
       │              ┌─────▼──────┐
       │              │   valkey   │
       ▼              │  (cache)   │
  ~/.hermes/          └────────────┘
  config.yaml
  skills/
  pip-packages/
```

---

## The Dual Backend Strategy

This repo supports two search backends, complementary:

| Backend | Setup | Stability | Results | When to use |
|---------|-------|:---------:|:-------:|-------------|
| **SearXNG** | Docker (included) | ⭐⭐⭐⭐⭐ | 17-20 | Default. Zero rate limits. |
| **ddgs** | `pip install ddgs` | ⭐⭐⭐ | 3-5 | Backup. No Docker needed. ~25% empty. |

Switch between them (edit `~/.hermes/config.yaml`):

```yaml
web:
  search_backend: "searxng"   # stable, recommended
  # search_backend: "ddgs"     # zero-infra fallback
```

---

## Web Extraction (web_extract)

SearXNG handles `web_search` but NOT `web_extract` (SearXNG doesn't return
full page content). The default extract backend is **Firecrawl**, which
ships with Hermes and offers a **keyless free tier** (1 000 credits/month, no
signup). It works out of the box — no API key needed.

If you hit the free tier limit or want higher quality extraction:

```bash
# Option A: Add a Firecrawl API key to ~/.hermes/.env
# (extract_backend: firecrawl is already the default)
echo 'FIRECRAWL_API_KEY=fc-your-key' >> ~/.hermes/.env

# Option B: Use the browser as fallback (always free, no key)
# The hermes-web-search-debugging skill documents browser_navigate as a
# drop-in replacement for web_extract on arxiv, docs.python.org, github, wikipedia.
```

Detailed diagnostics: load the `hermes-web-search-debugging` skill.

---

## Known Pitfalls (Read Before You Debug)

### 1. Silent Fallback Trap

If `search_backend: ddgs` but ddgs isn't installed, Hermes doesn't error — it
silently falls back to firecrawl. Your search still works, so you never notice
the config is wrong. Check which backend is actually active:

```python
from agent.web_search_registry import get_active_search_provider
print(get_active_search_provider().name)
```

### 2. ddgs LAZY_DEPS Gap

Hermes auto-installs SDKs for firecrawl, exa, and parallel. ddgs is the only
backend NOT in the auto-install allowlist. After Docker rebuild, ddgs disappears
silently. Fix: persist packages to `/opt/data/pip-packages/` + `PYTHONPATH`.

### 3. Cloudflare Blocks AI Agents

Since July 2025, Cloudflare blocks AI agent traffic. `browser_navigate` and
`web_extract` both fail silently on Cloudflare-protected sites (Indeed, Glassdoor,
Crunchbase, etc.). Use Scrapling with Playwright Chromium as fallback.

### 4. SearXNG Secret Key

The `secret_key` field in `settings.yml` has been intentionally left commented out.
SearXNG SHOULD auto-generate one on first startup, but this behavior depends on the
Docker entrypoint script version. If you encounter session errors or CSRF rejections
after restarting SearXNG, generate a fixed key:

```bash
# Add to docker-compose.yml under searxng-core → environment:
#   SEARXNG_SECRET: a94a8fe5ccb19ba61c4c0873d391e987982fbbd3
# (generate your own with: openssl rand -hex 32)
```

---

## What's NOT Included

- API keys (`.env`) — use your own
- Runtime data (`sessions/`, `memories/`, `state.db`)
- Project-specific skills (this is search-only)
- Playwright Chromium download (one-time ~114MB, see Quick Start)

---

## License

MIT
