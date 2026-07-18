# SearXNG Engine Reliability Reference

Updated 2026-07-17 after full diagnosis + fix of a self-hosted Docker instance
(hermes + searxng-core + searxng-valkey, foreign IP, WSL host).

## Engine Reliability Table (self-hosted Docker, foreign IP)

| Engine | Status | Notes |
|--------|--------|-------|
| **bing** | ✅ Reliable | Only web engine that survives long-term. Needs `timeout: 15.0` per engine + `outgoing.request_timeout: 10.0`. Default 3s is too short. |
| **mojeek** | ✅ Reliable | Independent index, stable alongside bing. ~10 results/query. Enable it. |
| brave | ❌ Suspended | Gets "too many requests" after weeks of use. Suspension is effectively permanent on same IP. |
| duckduckgo | ❌ CAPTCHA | Triggers on nearly all self-hosted instances. 3600s suspension timer, keeps re-triggering. |
| startpage | ❌ CAPTCHA | Same as duckduckgo. "CAPTCHA on all public instances" per Reddit r/Searx consensus. |
| presearch | ❌ Access denied | "Suspended: access denied" on self-hosted instances. Likely IP-based. |
| qwant | ❌ Access denied | Same as presearch. Blocked for self-hosted instances. |
| google | 🔒 API key | `inactive: true` in defaults. Requires Google Custom Search JSON API key. Not practical for self-hosted. |
| wikipedia | ✅ Always | No rate limiting. Always works. |
| arxiv | ✅ Always | Academic search. No rate limiting. |
| stackoverflow | ✅ Always | Programming Q&A. No rate limiting. |
| github/docker/pypi | ✅ Always | Code/package search. No rate limiting. |

## Fix Applied (2026-07-17)

settings.yml changes:
```yaml
outgoing:
  request_timeout: 10.0       # was 3.0

engines:
  - name: bing
    timeout: 15.0              # added — was missing, defaulted to 3s
    disabled: false

  - name: duckduckgo
    disabled: true             # was enabled — CAPTCHA'd

  - name: brave
    disabled: true             # was enabled — suspended

  - name: startpage
    disabled: true             # was enabled — CAPTCHA'd

  - name: mojeek
    disabled: false            # was disabled: true — now reliable second engine

  - name: presearch
    disabled: true             # was enabled — access denied
```

Then: `docker compose restart searxng-core`

## Result

| Metric | Before | After |
|--------|--------|-------|
| Working web engines | 1 (bing unreliable due to timeout) | 2 (bing + mojeek) |
| Results per query | 0–10 | 17–20 |
| Dead engines polluting searches | 4 | 0 (all disabled) |

## SearXNG Suspension Timers (from settings.yml)

```yaml
suspended_times:
  SearxEngineAccessDenied: 180
  SearxEngineCaptcha: 3600          # 1 hour
  SearxEngineTooManyRequests: 180    # 3 minutes
  cf_SearxEngineCaptcha: 1296000     # 15 days!
  cf_SearxEngineAccessDenied: 86400  # 24 hours
  recaptcha_SearxEngineCaptcha: 604800  # 7 days
```

## CAPTCHA Mitigation (official, requires external resources)

The only automated solution is proxy rotation via `outgoing.proxies`:
```yaml
outgoing:
  proxies:
    all://:
      - http://proxy1:8080
      - http://proxy2:8080
  using_tor_proxy: true
```

See: https://docs.searxng.org/admin/settings/settings_outgoing.html

Manual CAPTCHA solving (interactive, not automated):
https://docs.searxng.org/admin/answer-captcha.html

## Diagnosis Command

```bash
curl -s "http://searxng-core:8080/search?q=test&format=json" | python3 -c "
import sys,json
d=json.load(sys.stdin)
print(f'results={len(d.get(\"results\",[]))}')
print(f'unresponsive={d.get(\"unresponsive_engines\",[])}')
engines={}
for r in d.get('results',[]):
    e=r.get('engine','?')
    engines[e]=engines.get(e,0)+1
for e,c in sorted(engines.items()): print(f'  {e}: {c}')
"
```

## Deployment Pitfall: Settings Not Applied (New Container)

When running SearXNG in Docker with a custom `settings.yml`, the file must be
in the mounted config directory **before** the container starts. Docker Compose
creates the directory on first run but does NOT copy your settings file.

**Symptom**: SearXNG starts but uses default config — DuckDuckGo/Brave/Startpage
enabled, all failing with CAPTCHA/suspension errors.

**Fix**: Copy settings before first `docker compose up`:

```bash
mkdir -p docker-data/searxng/config
cp docker/searxng/settings.yml docker-data/searxng/config/
docker compose up -d
```

If the container already started without the settings file, copy it in and restart:

```bash
cp docker/searxng/settings.yml docker-data/searxng/config/
docker compose restart searxng-core
```
