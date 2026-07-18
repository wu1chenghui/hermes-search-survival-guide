# AGENTS.md — Hermes Search Survival Guide

AI agent context for this repository. Auto-loaded when `working_dir` points here.

## Quick Start (executable)

```bash
git clone <repo> && cd hermes-search-survival-guide
mkdir -p docker-data/searxng/config docker-data/searxng/cache ~/.hermes/skills
cp config/config.yaml.template ~/.hermes/config.yaml
cp docker/searxng/settings.yml docker-data/searxng/config/
cp -r skills/* ~/.hermes/skills/
docker compose up -d
```

To install ddgs and Scrapling (optional, for dual-backend setup):
```bash
docker compose exec hermes bash -c "uv pip install --target /opt/data/pip-packages ddgs h2 httpcore brotli fake-useragent lxml primp"
docker compose exec hermes bash -c "uv pip install --target /opt/data/pip-packages 'scrapling[all]'"
```

## Key Paths

| Path | Purpose |
|------|---------|
| `~/.hermes/config.yaml` | Hermes configuration (copy from `config/config.yaml.template`) |
| `~/.hermes/skills/` | Skills directory (copy from `skills/`) |
| `docker-data/searxng/config/settings.yml` | SearXNG engine configuration (copy from `docker/searxng/settings.yml`) |
| `docker-data/searxng/cache/` | SearXNG cache (auto-created) |
| `/opt/data/pip-packages/` | Persistent Python packages (Docker volume to `~/.hermes/`) |
| `/opt/data/.playwright/` | Persistent Playwright Chromium install |

## Environment Variables

Set in `docker-compose.yml` (NOT `.env` alone):

| Variable | Value | Required |
|----------|-------|:--------:|
| `HERMES_HOME` | `/opt/data` | Yes |
| `SEARXNG_URL` | `http://searxng-core:8080` | Yes |
| `PYTHONPATH` | `/opt/data/pip-packages` | For ddgs/Scrapling |

## Infrastructure Components

| Container | Image | Port | Purpose |
|-----------|-------|:----:|---------|
| hermes | nousresearch/hermes-agent:latest | 9119 | Main agent |
| searxng-core | searxng/searxng:latest | 8085→8080 | Search aggregation |
| searxng-valkey | valkey/valkey:9-alpine | — | SearXNG cache backend |

## Skills Included

| Skill | Location | Purpose |
|-------|----------|---------|
| hermes-web-search-debugging | skills/hermes-web-search-debugging/ | Diagnose search failures, all 8 backends |
| web-search-strategy | skills/web-search-strategy/ | Multi-round research methodology |
| searxng-config | skills/searxng-config/ | SearXNG engine config + diagnostics |

## Critical Warnings

1. **Only bing and mojeek work on self-hosted SearXNG.** brave, ddg, startpage, presearch, qwant are all permanently dead. The full settings.yml disables them; the minimal config only enables bing + mojeek + non-web engines.

2. **Silent fallback**: if `search_backend: ddgs` but ddgs not installed, Hermes silently falls back to firecrawl. Verify with `get_active_search_provider().name`.

3. **ddgs not in LAZY_DEPS**: Hermes auto-installs SDKs for firecrawl/exa/parallel but NOT ddgs. After container rebuild, ddgs disappears. Persist via `pip install --target /opt/data/pip-packages` + `PYTHONPATH`.

4. **SEARXNG_URL must be in docker-compose.yml**: a comment in `.env` is not enough. The agent reads from the container environment.

5. **Settings path**: after copying `docker/searxng/settings.yml` to `docker-data/searxng/config/`, edit in place. Restart SearXNG with `docker compose restart searxng-core`.

6. **Cloudflare blocks AI agents** (since July 2025). Use Scrapling (`solve_cloudflare=True`) with Playwright Chromium at `/opt/data/.playwright/` as fallback.

7. **Dual-backend pattern**: SearXNG (stable, 17-20 results, requires Docker) + ddgs (free, no infra, ~25% empty). Switch with `hermes config set web.search_backend <backend>`.
