# ddgs Version Quirks — Diagnostic Report (2026-07-11)

## Dependency Tree (known-good install)

```
ddgs 9.14.4
├── httpcore 1.0.9
│   └── h2 4.3.0 (optional — required for http2)
├── httpx 0.28.1
├── lxml, primp, brotli, fake-useragent
```

Install:
```bash
uv pip install ddgs h2 httpcore brotli fake-useragent lxml primp --python /opt/hermes/.venv/bin/python3
```

## Failure Mode 1: httpcore._sync has no attribute 'http2' (~5-15%)

**Root cause**: httpcore `_sync/__init__.py` does `try: from .http2 import HTTP2Connection except ImportError: ...` When the import fails (h2 missing or import error), the `except` creates a dummy `HTTP2Connection` class but does NOT add the `http2` submodule as an attribute. ddgs/http_client2.py L124 then accesses `httpcore._sync.http2.HTTP2Connection` → AttributeError.

**Reproduction**: Block h2 import → error is 100% reproducible. With h2 installed → 80 consecutive attempts → 0 failures. The error is therefore rare in practice but triggered when h2 import fails intermittently in Hermes' process model.

**Fix**: Retry the query. The import succeeds on the next attempt. Do NOT downgrade anything.

## Failure Mode 2: "No results found" (~25%) — PRIMARY FAILURE

**Not a timeout**. The ddgs library's HTTP request completes but DuckDuckGo returns an empty HTML page. This is server-side rate-limiting or anti-bot detection.

**Characteristics**:
- OK responses: median 12s latency, range 2-19s
- "No results": ~25% of queries, DDGSException raised
- Academic queries (mathematics, Lie algebra) are faster (3-6s) than generic queries (12-19s)

**Workaround**: 
- Increase `DDGS(timeout=20)` — gives server more time
- Add 3-5s sleep between queries — reduces rate-limit triggering
- Prefer specific queries over generic ones

## Failure Mode 3: Genuine timeouts (~5%)

Rare. Some queries take >15s and time out. Increasing timeout to 20-30s mitigates this.

## Result Format

ddgs returns dicts with keys: `title`, `href`, `body`. The Hermes provider wrapper maps `href` → `url`. Not `description` — the body text IS the description.

## 9.14.4 (current, 2026-07-08)

### Dependencies
```
ddgs==9.14.4
├── brotli==1.2.0
├── fake-useragent==2.2.0
├── h2==4.3.0
├── hpack==4.2.0
├── hyperframe==6.1.0
├── httpcore==1.0.9
├── httpx==0.28.1
├── lxml==6.1.1
└── primp==1.3.1
```

### Install command (uv, since pip unavailable)
```bash
uv pip install ddgs h2 httpcore brotli fake-useragent lxml primp \
  --python /opt/hermes/.venv/bin/python3
```

### Result format
ddgs 9.14.4 returns dicts with keys `title`, `href`, `body` (NOT `url`).
Hermes' ddgs provider in `plugins/web/ddgs/provider.py` maps `href` → `url`.

### Primary failure mode (2026-07-11 diagnosis)

ddgs 9.14.4 + httpcore 1.0.9 + h2 4.3.0. Benchmarked over 30 calls:

| Outcome | Rate | Latency |
|---------|------|---------|
| OK (returns results) | ~70% | median 12s, range 2-19s |
| "No results found" | ~25% | varies |
| http2 AttributeError | <1% | N/A |
| Actual timeout | <5% | N/A |

**Primary issue is "No results found" + high latency**, not http2 errors.
DuckDuckGo rate-limits automated queries; delays of 10-19s are normal.

### Intermittent httpcore error (RARE — <1%)
- Symptom: `module 'httpcore._sync' has no attribute 'http2'`
- **Reproducible root cause** (confirmed 2026-07-11): when `h2` package import fails, `httpcore/_sync/__init__.py`'s `except ImportError` creates a dummy `HTTP2Connection` class but does NOT add `http2` as a submodule attribute. ddgs `http_client2.py:124` then accesses `httpcore._sync.http2.HTTP2Connection` → AttributeError
- **Not reproducible in normal operation**: 80 test calls (sequential, concurrent 10-thread, forced reimport ×50) → 0 occurrences. Only observed in Hermes process context (possible fork/subprocess module state inconsistency)
- Fix: retry the query — import succeeds next attempt. Do NOT pin/downgrade httpcore or ddgs

### Search engines used
ddgs 9.14.4 tries engines in this order:
1. DuckDuckGo (primary, via `html.duckduckgo.com/html/`)
2. Mojeek (fallback)
3. Other engines (Brave, Bing, etc.)

When DuckDuckGo times out, it falls back to Mojeek. Error messages may
reference Mojeek URLs even though the primary backend is DuckDuckGo.

### Direct test command
```bash
/opt/hermes/.venv/bin/python3 -c "
from ddgs import DDGS
d = DDGS()
results = list(d.text('test', max_results=3))
print(f'Got {len(results)} results')
[print(f'  {r[\"title\"][:80]}') for r in results]
"
```
Expected: ≥1 result with titles.

### web_extract with ddgs
ddgs supports search only (`supports_extract() → False`). For extraction,
configure a separate extract backend (firecrawl, tavily, exa, parallel) or
use `browser_navigate` → `browser_snapshot(full=true)` as a workaround.
