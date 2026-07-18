---
name: searxng-config
description: >-
  Configure SearXNG for self-hosted instances. Engine reliability table,
  minimal working config, and engine-by-engine diagnostics. Standalone —
  does not require Hermes.
version: 1.0.0
---

# SearXNG Configuration Guide

## When to load

- Setting up a new SearXNG instance
- web_search returns empty results from SearXNG
- Need to know which engines actually work on self-hosted instances

## Engine Reliability Table

After extensive testing on a self-hosted SearXNG instance (2026-07):

| Engine | Status | Results | Action |
|--------|:------:|:-------:|--------|
| **bing** | ✅ | ~10 | Enable. Needs `timeout: 15.0`. |
| **mojeek** | ✅ | ~9-10 | Enable. Independent index, rarely blocked. |
| brave | ❌ | 0 | **Disable.** Suspended: too many requests. Permanent ban on self-hosted. |
| duckduckgo | ❌ | 0 | **Disable.** CAPTCHA on all self-hosted instances. |
| startpage | ❌ | 0 | **Disable.** CAPTCHA on all self-hosted instances. |
| presearch | ❌ | 0 | **Disable.** Access denied. |
| qwant | ❌ | 0 | **Disable.** Access denied. |
| google | 🔒 | 0 | Requires Google Custom Search JSON API key (paid). |
| wikipedia | ✅ | varies | Enable. Never CAPTCHA'd. |
| arxiv | ✅ | varies | Enable. Never CAPTCHA'd. |
| github | ✅ | varies | Enable. Never CAPTCHA'd. |
| stackoverflow | ✅ | varies | Enable. Never CAPTCHA'd. |
| docker hub | ✅ | varies | Enable. Never CAPTCHA'd. |
| wikidata | ✅ | varies | Enable. Never CAPTCHA'd. |
| bing images | ✅ | varies | Enable. |
| bing news | ✅ | varies | Enable. |

**Key insight**: brave, ddg, startpage, presearch, and qwant are universally blocked
on self-hosted SearXNG. This is a business decision by these search providers, not a
configuration bug. Do not waste time debugging them — disable and move on.

## Minimal Working Config

Copy `templates/settings.yml` to `docker-data/searxng/config/settings.yml`.

It enables only: bing, mojeek, wikipedia, arxiv, github, docker hub, stackoverflow,
wikidata, bing images, bing news.

## Engine Diagnose Commands

### Check which engines are responding

```bash
curl -s --max-time 30 "http://searxng-core:8080/search?q=test&format=json" | python3 -c "
import sys,json
d=json.load(sys.stdin)
print(f'Total results: {len(d.get(\"results\",[]))}')
print(f'Unresponsive engines: {d.get(\"unresponsive_engines\",[])}')
engines={}
for r in d.get('results',[]):
    e=r.get('engine','?')
    engines[e]=engines.get(e,0)+1
for e,c in sorted(engines.items()): print(f'  {e}: {c}')
"
```

From host (outside Docker):
```bash
curl -s "http://localhost:8085/search?q=test&format=json" | python3 -c "..."
```

### Test a single engine

```bash
curl -s --max-time 30 "http://searxng-core:8080/search?q=test&format=json&engines=mojeek" | python3 -c "
import sys,json
d=json.load(sys.stdin)
print(f'Results: {len(d.get(\"results\",[]))}')
print(f'Unresponsive: {d.get(\"unresponsive_engines\",[])}')
"
```

### After editing settings.yml

```bash
docker compose restart searxng-core
# Then verify:
curl -s --max-time 30 "http://searxng-core:8080/search?q=test&format=json" | python3 -c "..."
```

## Common Pitfalls

1. **JSON format disabled.** SearXNG disables JSON output by default. Add `json` to
   `search.formats` in settings.yml.
2. **bing timeout.** Default `request_timeout: 3.0` is too low for bing. Set
   `outgoing.request_timeout: 10.0` and per-engine `timeout: 15.0`.
3. **secret_key in settings.yml.** If using `use_default_settings: true`, any
   `secret_key` in your local settings.yml is ignored — SearXNG uses the one from
   the base config. Either remove it or use the `SEARXNG_SECRET` env var.
4. **Engine not in `engines` list.** `use_default_settings: true` pulls in all
   default engines. You must explicitly `disabled: true` the ones you don't want.
   Our minimal config uses `use_default_settings: true` + explicit disables.
