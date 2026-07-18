# Lazy Deps Coverage Gap: ddgs

## Discovery (2026-07-17)

Hermes has a lazy dependency installer (`tools/lazy_deps.py`) that auto-installs
Python packages when a backend provider is first used. The mechanism uses an
allowlist (`LAZY_DEPS` dict) and each provider calls `ensure("feature.name")`.

## Coverage

| Backend | LAZY_DEPS key | Provider calls ensure() | Auto-installs? |
|---------|:---:|:---:|:--:|
| firecrawl | `search.firecrawl` | ✅ provider.py:83 | ✅ |
| exa | `search.exa` | ✅ provider.py:64 | ✅ |
| parallel | `search.parallel` | ✅ provider.py:57 | ✅ |
| **ddgs** | *(absent)* | ❌ | ❌ |

## Implications

- ddgs is the only one of the 8 web search backends not covered by lazy_deps.
- After Docker container rebuild, ddgs must be manually reinstalled:
  ```
  uv pip install ddgs h2 httpcore brotli fake-useragent lxml primp \
    --python /opt/hermes/.venv/bin/python3
  ```
- The other three backends (firecrawl/exa/parallel) auto-recover via lazy_deps
  on first use.

## Why This Matters

The official docs say "or let Hermes lazy-install it on first use" for DDGS,
but this is only true on bare-metal installs where `hermes tools` post-setup has
run. In Docker, the post-setup must be triggered manually or the package must be
pip-installed by hand.

## Hermes' lazy_deps Found Elsewhere

Lazy deps covers 20+ features beyond web search: TTS providers (mistral, edge,
elevenlabs), STT (mistral, faster_whisper), image gen (fal), memory providers
(honcho, hindsight), messaging platforms (telegram, discord, slack, matrix,
dingtalk, feishu), terminal backends (modal, daytona), and tools (ACP, dashboard,
vision). All use the same `ensure()` pattern with exact-pinned versions matched
to pyproject.toml.
