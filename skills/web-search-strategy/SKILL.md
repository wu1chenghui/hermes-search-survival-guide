---
name: web-search-strategy
description: >-
  Search strategy for web research tasks. TRIGGER on: "search for X",
  "research X", "look up X", "find information about X", "what is X",
  "compare X and Y", or any task that needs web information. SKIP on:
  single fact lookup (just call web_search directly), code debugging,
  file operations. Use this BEFORE starting any content generation
  that requires real-world data.
version: 1.0.0
metadata:
  hermes:
    tags: [research, search, web, strategy]
    requires_toolsets: [web, browser]
---

# Web Search Strategy

Systematic multi-round web search methodology. Load this when the task needs
more than one web_search call — any research question, comparison, or content
generation that requires real-world information.

## Core Principle

**A single web_search is almost never enough.** The quality of your output
depends on how thoroughly you searched before generating. One query → one
perspective → shallow answer.

## When to Use

**Always use when:**
- User asks "research X", "find information about X", "what is X"
- User wants comparison, analysis, or multi-perspective answers
- Before generating any content (articles, reports, summaries) that needs facts
- The topic has multiple dimensions (technical, historical, market, etc.)

**Skip when:**
- Single fact lookup (e.g. "what version is Python 3.13") → just call web_search
- Code debugging or implementation → no search needed
- The answer is in your training data and needs no verification

## Our Search Backends

| Backend | Status | Characteristics |
|---------|:------:|-----------------|
| **ddgs** | ✅ Available | Free, no key. ~25% empty-result rate. 3-5 results/query. Add 10s delay between calls. Persisted via PYTHONPATH → `/opt/data/pip-packages/`. Not in LAZY_DEPS (ddgs is the only search backend uncovered by Hermes' auto-installer). |
| **SearXNG** | ✅ Available | Self-hosted Docker at searxng-core:8080. bing + mojeek engines, 17-20 results/query, no rate limit. Free, no key. Switch via `hermes config set web.search_backend searxng`. |
| **browser** | ✅ Always | Full page extraction. Use when search snippets are insufficient. 3-5s per page. Cloudflare-blocked pages show "Just a moment..." — skip or use Scrapling. |
| **Scrapling** | ✅ Ready | Stealth browser with Cloudflare bypass (`solve_cloudflare=True`). Playwright Chromium at `/opt/data/.playwright/`. Python package at `/opt/data/pip-packages/`. Use when browser_navigate hits a block wall. Adds 5-15s. |
| **web_extract** | ❌ Unavailable | No extract backend configured. Use browser instead. |

**When to switch to SearXNG**: If ddgs returns empty results on 2 consecutive calls,
or if the task requires >5 web_search calls. The SearXNG backend delivers more
results with zero rate-limiting.

## The 4-Phase Method

### Phase 1: Broad Exploration (2-3 searches)

Map the territory. Don't read anything yet — just collect promising URLs.

1. Search the main topic with a general query
2. From results, identify 2-4 key dimensions/subtopics
3. Search each dimension with targeted queries

**Query construction:**
```
✅ "enterprise AI adoption trends 2026"
✅ "transformer architecture explained research paper"
✅ "finite simple groups classification theorem proof"

❌ "AI trends"                          ← too vague
❌ "what is a transformer"              ← too basic, waste of a search
```

**After this phase**: you should have a list of 5-15 candidate URLs across
different sources.

### Phase 2: Deep Read (browser extraction)

For the 3-5 most authoritative/relevant URLs from Phase 1, extract full content:

```
browser_navigate(url="https://target-page.com")
browser_snapshot(full=true)
```

**Cloudflare / anti-bot blocking**: Since July 2025, Cloudflare blocks AI agents
by default. The block is silent — the page returns a 200 with a "Just a moment..."
body that looks like real content. Always check the snapshot title after navigating.

**Detection + skip (primary):**
- If the page title is `"Just a moment..."` or snapshot shows only a security check → the page is blocked
- **Do NOT read the block page as content.** Report: "This page is Cloudflare-blocked."
- Move to the next URL in your candidate list.

**Scrapling stealth fetch (fallback, when the page is critical):**
```bash
PLAYWRIGHT_BROWSERS_PATH=/opt/data/.playwright \
  /opt/hermes/.venv/bin/scrapling extract stealthy-fetch '<URL>' output.md \
  --solve-cloudflare --block-webrtc --hide-canvas
```
Then read `output.md` with read_file. Playwright Chromium 149 installed at
`/opt/data/.playwright/`. Adds 5-15 seconds per page — only use when the page
is essential and browser_navigate was blocked.

> **Full anti-bot solution reference**: `references/anti-bot-blocking.md` —
> benchmark data, Scrapling/Camofox/Browserbase comparison, pass rates.

**Source credibility order:**
1. Official docs, government/institutional sites (.gov, .edu, docs.*.org)
2. Wikipedia, established reference sites
3. Authoritative tech blogs, research group pages
4. Community discussions (GitHub issues, StackOverflow) — for context, not facts

**Do NOT** rely on search snippets alone for important claims. Snippets are
often truncated, decontextualized, or plain wrong.

### Phase 3: Gap Check & Validation (1-3 searches)

After Phase 1-2, ask yourself:

- Do I have concrete data/numbers, not just descriptions?
- Have I seen this from 2+ independent sources?
- Is there a counter-argument or limitation I haven't found?
- Would a search from the opposite perspective find something different?

Run gap-filling searches. Seek disagreement — the best research finds what
contradicts the consensus.

### Phase 4: Deliver

Synthesize findings. Always attribute claims to sources. If a claim comes from
a search snippet, say so. If from a fully-read page, cite it.

## Search Budget Control

| Task complexity | Max searches | When to stop |
|-----------------|:-----------:|--------------|
| Simple lookup | 1-2 | Got the answer |
| Medium research | 5-10 | 2+ independent sources agree |
| Deep research | 15-25 | All dimensions covered + gap check passed |

**Hard stop rule**: if 3 consecutive ddgs calls return empty results, switch to
SearXNG or report the blocker. Do NOT enter a retry loop.

**Delay rule**: when using ddgs, add 10 seconds between successive web_search
calls. Reduces DuckDuckGo rate-limiting and empty-result rate. Use
`execute_code` with `time.sleep(10)` between calls.
`execute_code` with `time.sleep(10)` between calls.

## Browser Extraction Tips

When search snippets aren't enough:
- `browser_navigate(url)` then `browser_snapshot(full=true)` — gets full page
- **Check for Cloudflare blocking**: if title is "Just a moment..." → skip this URL
- For dynamic pages: `browser_scroll(direction="down")` before snapshot
- For structured data: `browser_console(expression="document.querySelector('article').innerText")`
- Verified working on: arxiv.org, docs.python.org, github.com, wikipedia.org
- Cloudflare-blocked sites (Indeed, Glassdoor, Crunchbase, etc.): use Scrapling stealth
  mode as fallback, or skip and use the next search result

Report progress between rounds: "Round 1 done — found 12 candidates across 3
dimensions. Now Round 2: deep-reading the top 4."

## Common Mistakes

- ❌ Calling web_search once and calling it done
- ❌ Reading only snippets without extracting full pages
- ❌ Searching only one phrasing of a question
- ❌ Blind retry on empty ddgs results without switching backend
- ❌ Using web_extract (it's unavailable — use browser)
- ❌ **Reading a Cloudflare \"Just a moment...\" block page as if it were real content** — always check the page title after browser_navigate
- ❌ Searching in Chinese when English would return more authoritative sources
  (or vice versa)
- ❌ **Starting searches without presenting a plan to the user** — design the
  plan, show the query table, get approval, then execute

## Quick Reference: Search Plan Design

For complex tasks, design your search plan BEFORE executing. **Present the plan
to the user for approval** — show the query table with rounds and goals,
then wait for the user to confirm before firing any searches.

```
Round 1: Broad survey         → 2-3 queries, different angles
Round 2: Targeted deep-dive    → 3-5 queries, specific dimensions
Round 3: Gap fill              → 1-3 queries, counter-perspectives
Round 4: Source extraction     → browser_navigate on top 3-5 URLs
Round 5: Final validation      → verify key claims with cross-reference
```

**Plan format** (present to user as a table):

| Round | Query | Goal |
|---|---|---|
| Round 1 | `"hermes agent" dotfiles config template github` | Find Hermes config repos |
| | `AGENTS.md spec recommended sections` | AGENTS.md standard |
| Round 2 | ... | ... |

Include search budget (max searches, hard-stop rule) and the delay between
calls. Do NOT execute any searches until the user approves the plan.

## Docker Container Persistence

We run inside a Docker container. The venv at `/opt/hermes/.venv/` is part of
the image and lost on rebuild. Extra Python packages must be persisted on the
mounted volume (`/opt/data/`).

**Pattern: `pip install --target` + `PYTHONPATH`** (same as Playwright's
`PLAYWRIGHT_BROWSERS_PATH`):

```bash
# One-time setup:
mkdir -p /opt/data/pip-packages
uv pip install --target /opt/data/pip-packages \
  ddgs h2 httpcore brotli fake-useragent lxml primp
uv pip install --target /opt/data/pip-packages \
  "scrapling[all]"
```

**docker-compose.yml** (add to `environment`):
```yaml
environment:
  - PYTHONPATH=/opt/data/pip-packages
```

**Why this works**: `/opt/data/` is a Docker volume mounted from `~/.hermes`.
Packages in `/opt/data/pip-packages/` survive container deletion. `PYTHONPATH`
prepends this directory to Python's import path, taking priority over the venv.

**ddgs LAZY_DEPS gap**: Hermes has a lazy dependency installer (`tools/lazy_deps.py`)
that auto-installs packages for firecrawl, exa, and parallel when they're first used.
ddgs is the ONLY search backend NOT in this allowlist. Tracking this gap so we don't
waste time debugging why ddgs doesn't auto-recover after rebuild.

**Playwright Chromium** is separately persisted at `/opt/data/.playwright/` via
`PLAYWRIGHT_BROWSERS_PATH`. The 114MB download is one-time.
