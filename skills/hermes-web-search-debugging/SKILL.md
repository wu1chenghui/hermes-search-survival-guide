---
name: hermes-web-search-debugging
description: >-
  Diagnose and fix Hermes web_search / web_extract failures.
  Covers all 8 backends (firecrawl, searxng, brave-free, ddgs, tavily,
  exa, parallel, xai). Step-by-step triage: silent-fallback detection →
  config → env var → plugin registration → upstream engine health.
  Includes SearXNG engine reliability table (tested 2026-07-17) and
  our actual deployment paths.
---

# Hermes Web Search Debugging

## When to load

- `web_search` or `web_extract` returns empty results or errors
- Hermes reports "set up a provider" or "SearXNG is a search-only backend"
- After changing `.env` or `config.yaml` web settings, search still fails
- **Search appears to work but you're unsure WHICH backend is actually handling requests**

## ⚠️ Silent Fallback Pitfall (READ FIRST)

**The configured backend and the active backend can silently diverge.** This is the
most common source of "search works but I don't know why" confusion.

When `web.search_backend` (or `web.backend`) names a backend that isn't actually
available (e.g. `ddgs` configured but `ddgs` package not installed, or `searxng`
configured but `SEARXNG_URL` not set), Hermes does NOT surface an error. Instead:

1. `_get_search_backend()` → `_is_backend_available("ddgs")` → False → falls through
2. `_get_backend()` → walks the candidate list → finds first available → returns it
3. If nothing available, returns `"firecrawl"` as the hardcoded default
4. The firecrawl plugin picks up the request (possibly via Tool Gateway, no key needed)

**Result: the user sees working search, thinks ddgs is handling it, but firecrawl is
actually doing the work.** This masks misconfiguration indefinitely.

**Detection**: check `get_active_search_provider().name` — if it doesn't match your
configured backend, you're in silent-fallback territory.

**Fix**: either install the configured backend's dependencies, or change config to
match what's actually available.

## Triage Order (always follow this sequence)

### 1. Check config keys

```bash
grep -A3 "^web:" /opt/data/config.yaml
```

Three keys matter:
- `web.backend` — shared fallback for both search and extract
- `web.search_backend` — per-capability override for search
- `web.extract_backend` — per-capability override for extract

**Empty string `''` is NOT the same as unset.** The legacy fallback chain
(firecrawl → parallel → tavily → exa → searxng → brave-free → ddgs) only
fires when the key is absent. An explicit `''` short-circuits it.

Fix: `hermes config set web.search_backend searxng`

### 2. Check env var in agent process

The agent reads `.env` at startup. `/reload` reloads `.env` but does
NOT re-discover plugins.

Verify SEARXNG_URL (or FIRE/CRAWL/TAVILY/EXA/PARALLEL key) is actually
in the agent's environment:

```python
import os
print(repr(os.getenv("SEARXNG_URL")))
```

If it's `None` but present in `.env`, the session was started before
the var was added. `/reset` (new session) is the fix — `/reload` alone
is insufficient because plugins were already discovered without the var.

### 3. Check plugin registration

```python
from agent.web_search_registry import list_providers, get_active_search_provider
providers = list_providers()
for p in providers:
    print(f"  {p.name}: available={p.is_available()}, search={p.supports_search()}, extract={p.supports_extract()}")
active = get_active_search_provider()
print(f"Active: {active.name if active else None}")
```

0 providers = plugins never loaded. Force reload in the same session:

```python
from hermes_cli.plugins import PluginManager
pm = PluginManager()
pm.discover_and_load(force=True)
```

But this only helps within `execute_code` — the agent process keeps
its own registry. Full fix: `/reset`.

### 4. Test SearXNG directly (if using SearXNG)

```bash
curl -s --max-time 30 "http://searxng-core:8080/search?q=test&format=json" | python3 -c "
import sys,json
d=json.load(sys.stdin)
print(f'results={len(d.get(\"results\",[]))}')
print(f'unresponsive={d.get(\"unresponsive_engines\",[]))}')
engines={}
for r in d.get('results',[]):
    e=r.get('engine','?')
    engines[e]=engines.get(e,0)+1
for e,c in sorted(engines.items()): print(f'  {e}: {c}')
"
```

On the host machine (outside the Hermes container), use the port-mapped URL:
```bash
curl -s "http://localhost:8085/search?q=test&format=json" | python3 -c "..."
```

### 5. Diagnose upstream engine failures

SearXNG aggregates results from multiple search engines. If all return
empty, check `unresponsive_engines` in the JSON response. Common patterns:

| Engine | Typical Failure | Root Cause |
|--------|----------------|------------|
| brave | Suspended: too many requests | Permanent rate-limit ban on self-hosted instances |
| duckduckgo | CAPTCHA | IP flagged — universal on self-hosted instances |
| startpage | CAPTCHA | IP flagged — universal on self-hosted instances |
| google | inactive: true in defaults | Requires Google Custom Search JSON API key — not IP-related |
| presearch | Suspended/access denied | Blocked for self-hosted instances |
| qwant | access denied | Blocked for self-hosted instances |
| bing | timeout | `request_timeout` too low (default 3s); set `outgoing.request_timeout: 10.0` + per-engine `timeout: 15.0` |
| mojeek | (reliable) | Independent index, rarely blocked — enable as second web engine |

**Key insight**: brave/ddg/startpage/presearch/qwant are universally blocked on
self-hosted SearXNG. Don't waste time debugging them — just disable and move on.
bing + mojeek are the only reliable web search engines for self-hosted instances.
Non-web engines (wikipedia, arxiv, stackoverflow, github, docker hub, etc.) never
trigger CAPTCHA and should all be enabled.

### 6. Fix SearXNG engine config

Settings path: `docker-data/searxng/config/settings.yml` (Docker volume
mounted from the repo's docker-compose.yml). The full 2782-line settings.yml
is included in this repo at `docker/searxng/settings.yml`. A minimal working
version is at `skills/searxng-config/templates/settings.yml`.

**Engine reliability (tested 2026-07-17):**

| Engine | Status | Action |
|--------|:------:|--------|
| bing | ✅ 10 results | Keep. Needs `timeout: 15.0`. |
| mojeek | ✅ 9-10 results | Keep. Enable if disabled. |
| brave | ❌ Suspended | **Disable.** Permanent ban. |
| duckduckgo | ❌ CAPTCHA | **Disable.** Universal on self-hosted. |
| startpage | ❌ CAPTCHA | **Disable.** Universal on self-hosted. |
| presearch | ❌ access denied | **Disable.** |
| qwant | ❌ access denied | **Disable.** |
| google | 🔒 needs API key | Skip. Requires Google Custom Search JSON API. |
| wikipedia/arxiv/github/etc. | ✅ | Enable all. Never CAPTCHA'd. |

**Minimal working config snippet:**

```yaml
outgoing:
  request_timeout: 10.0

engines:
  - name: bing
    engine: bing
    shortcut: bi
    disabled: false
    timeout: 15.0

  - name: mojeek
    engine: mojeek
    shortcut: mjk
    categories: [general, web]
    disabled: false

  - name: brave
    disabled: true
  - name: duckduckgo
    disabled: true
  - name: startpage
    disabled: true
  - name: presearch
    disabled: true
  - name: qwant
    disabled: true
```

**After editing**: `docker compose restart searxng-core`

**Verify**: `curl -s --max-time 30 "http://searxng-core:8080/search?q=test&format=json"` → expect 17-20 results from bing + mojeek, unresponsive list empty or presearch only.

### 7. Verify end-to-end after fix

After fixing config and `/reset`:
```python
# Should return non-empty results
web_search("test mathematics")
```

## Browser Infrastructure

Hermes bundles its own headless Chromium via the `agent-browser` npm package.
Chrome 149 (Chrome for Testing) lives at `/opt/data/.agent-browser/browsers/`
and runs with `--headless=new --no-sandbox`. `browser_navigate` works immediately
with no setup.

> Full paths and architecture: `references/browser-infrastructure.md`

## Key Distinction: `/reload` vs `/reset`

| Command | Reloads .env | Re-discovers plugins | Fixes search |
|---------|:---:|:---:|:---:|
| `/reload` | ✅ | ❌ | Only if plugins OK, env was missing |
| `/reset` | ✅ | ✅ | ✅ Always |

Rule: if you change `.env` or `config.yaml` web keys, `/reset` is the
safe choice.

## ddgs LAZY_DEPS Gap (Docker-specific)

Hermes has a lazy dependency installer (`tools/lazy_deps.py`) that auto-installs
SDK packages when a backend is first used. The allowlist covers:

- `search.exa`, `search.firecrawl`, `search.parallel` — ✅ auto-install on first use
- `search.ddgs` — ❌ **NOT in allowlist**

In Docker, this means ddgs is lost on container rebuild and NOT auto-recovered.
The other three backends auto-recover. Fix: either `uv pip install ddgs` manually
after rebuild, or persist packages to the mounted volume via `pip install --target
/opt/data/pip-packages` + `PYTHONPATH` (see `web-search-strategy` skill).

## SearXNG Container Access

SearXNG runs as `searxng-core` container defined in `docker-compose.yml`.
Docker network URL: `http://searxng-core:8080` (from inside Hermes container).
Host-mapped URL: `http://localhost:8085` (from host machine).
Settings: `docker-data/searxng/config/settings.yml`.

**Critical: `SEARXNG_URL` must be in docker-compose.yml, not just `.env`.** When running
in Docker, set it as a container environment variable:

```yaml
services:
  hermes:
    environment:
      - SEARXNG_URL=http://searxng-core:8080
```

A comment-only entry in `.env` is NOT sufficient — the agent process reads env vars
from the container environment, not from commented-out lines.

The official SearXNG docs have a complete docker-compose.yml + settings.yml walkthrough
at https://hermes-agent.nousresearch.com/docs/user-guide/features/web-search.

JSON format must be enabled explicitly (it's disabled by default in SearXNG):

```yaml
search:
  formats:
    - html
    - json
```

### 8. Diagnose ddgs backend

> **Version-specific details** in `references/ddgs-version-quirks.md` —
> dependency tree, install commands, result format, and known race conditions.

When `search_backend: ddgs`, the `ddgs` Python package must be installed
in the Hermes venv. Test directly:

```bash
# Use the venv's own Python (not system python3)
/opt/hermes/.venv/bin/python3 -c "
from ddgs import DDGS
d = DDGS()
results = list(d.text('test', max_results=2))
print(f'Got {len(results)} results')
[print(r['title'][:80]) for r in results]
"
```

Expected: ≥1 result with titles. If the import fails (`ModuleNotFoundError`)
or returns 0 results:

Install fix (ddgs + its transitive deps, using uv since pip may not be available):
```bash
uv pip install ddgs h2 httpcore brotli fake-useragent lxml primp --python /opt/hermes/.venv/bin/python3
```

Verify with the venv's own Python (not system python3 — different site-packages):
```bash
/opt/hermes/.venv/bin/python3 -c "
from ddgs import DDGS
d = DDGS()
results = list(d.text('test', max_results=3))
print(f'Got {len(results)} results')
[print(f'  {r[\"title\"][:80]}') for r in results]
"
```

Expected: ≥1 result with titles. Result dicts use keys `title`, `href`, `body`
(not `url` — the provider wrapper maps `href` → `url`).

Restart note: ddgs package install is usually picked up immediately without
restart (Python imports are dynamic). A full agent restart is only needed
when config.yaml ownership/permissions changed (e.g. after `hermes config set`
rewrites it as root:640). Check with `ls -l /opt/data/config.yaml`.

Quick smoke test from within a session:
```
web_search("python")  # should return ≥1 result
```

**Important: ddgs is NOT in Hermes' lazy_deps allowlist.** Unlike firecrawl, exa,
and parallel (which auto-install their SDKs via `tools/lazy_deps.ensure()`), ddgs
must be manually installed. After container rebuild, ddgs disappears and needs
reinstall. Full analysis in `references/lazy-deps-coverage.md`.

### 8a. Diagnose ddgs failure modes (2026-07-11 update)

Three independent failure modes identified. See `references/ddgs-version-quirks.md`
for root cause analysis, reproduction steps, and workarounds.

**Quick summary:**

| Mode | Rate | Symptom | Fix |
|------|------|---------|-----|
| http2 AttributeError | ~5% | `httpcore._sync has no attribute 'http2'` | Retry — import race, succeeds next attempt |
| No results found | ~25% | `DDGSException("No results found")` — server returns empty page | Longer timeout (20s), 3-5s inter-query sleep |
| Genuine timeout | ~5% | >15s with no response | Increase timeout to 20-30s |

ddgs returns dicts with keys `title`, `href`, `body`. Hermes maps `href` → `url`.

**Primary failure is NOT timeouts or http2 errors — it's DuckDuckGo returning
empty pages (~25%) due to rate-limiting/anti-bot detection.**

**User preference (2026-07-11):** When running multiple web_search calls in
succession, add 3–5 second delays between them and use longer timeouts
(≥20s). This reduces the "No results" rate from DuckDuckGo rate-limiting.
Do NOT retry immediately on empty results — wait, then retry once. Stop after
two consecutive empty-result failures and switch to browser or report the
blocker honestly.

## Backend Capabilities (8 providers)

| Backend | Search | Extract | Free tier | Auth Required |
|---------|:---:|:---:|------|:---:|
| firecrawl (default) | ✅ | ✅ | 1 000 credits/mo (keyless) | FIRECRAWL_API_KEY (optional) |
| searxng | ✅ | ❌ | Free (self-hosted) | SEARXNG_URL |
| brave-free | ✅ | ❌ | 2 000 queries/mo | BRAVE_SEARCH_API_KEY |
| ddgs | ✅ | ❌ | Free (no key) | `ddgs` pip package |
| tavily | ✅ | ✅ | 1 000 searches/mo | TAVILY_API_KEY |
| exa | ✅ | ✅ | 1 000 searches/mo | EXA_API_KEY |
| parallel | ✅ | ✅ | Paid | PARALLEL_API_KEY |
| xai (Grok) | ✅ | ❌ | Paid | XAI_API_KEY or OAuth |

Firecrawl is the default and ships with Hermes. As of June 2026 it offers a keyless
free tier (1 000 credits/mo, no signup). SearXNG and Brave/DDGS are search-only —
pair with firecrawl/tavily/exa/parallel for `web_extract`.

**Per-capability split is the recommended pattern** (from official docs): use a free
search backend (SearXNG/DDGS/Brave) + a paid extract backend (Firecrawl):

```yaml
web:
  search_backend: "searxng"     # free search
  extract_backend: "firecrawl"  # high-quality extraction
```

When per-capability keys are empty, both fall through to `web.backend`. When
`web.backend` is also empty, auto-detection fires (firecrawl → parallel → tavily →
exa → searxng). xAI and ddgs are NOT in the auto-detection chain — opt in explicitly.

SearXNG is search-only. For `web_extract` you need firecrawl, tavily,
exa, or parallel — each requiring its own API key.

### 9. Browser as web_extract substitute

When `extract_backend` is not configured and no API-key extract backend
is available, use the browser toolchain as a drop-in replacement:

```
browser_navigate(url="https://target-page.com")
browser_snapshot(full=true)
```

**Verified on these site types (2026-07-08):**

| Site | Works? | Notes |
|------|:---:|-------|
| arxiv.org/abs/* | ✅ | Abstracts, LaTeX math, author lists — full extraction |
| docs.python.org | ✅ | Code blocks, TOC, structured API docs |
| github.com/* | ✅ | File listings, commit history, repo metadata |
| wikipedia.org | ✅ | Handles redirects, structured sections |

**What the browser can do that web_extract can't:**
- JavaScript-rendered content (SPAs, dynamic pages)
- Interactive navigation (follow links, click tabs, scroll for lazy-loaded content)
- Cookie walls and login gates (via `browser_click`/`browser_type`)
- Page screenshots (via `browser_screenshot`)

**Trade-offs vs web_extract:**
- Slower: each navigate ~3–5s vs web_extract ~1–2s
- Heavier: full browser session per navigation
- No built-in summarization: you get raw accessibility-tree content

**When to prefer web_extract over browser:**
- Batch extraction (5+ URLs in parallel via `web_extract(urls=[...])`)
- Simple text pages where speed matters
- When you already have a configured extract_backend with an API key
