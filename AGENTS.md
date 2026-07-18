# AGENTS.md — Hermes Search Survival Guide

AI agent context for this repository. Auto-loaded when `working_dir` points here.

## Quick Start (executable)

```bash
git clone <repo> && cd hermes-search-survival-guide
mkdir -p docker-data/searxng/config docker-data/searxng/cache ~/.hermes/skills
cp docker/searxng/settings.yml docker-data/searxng/config/
cp -r skills/hermes-web-search-debugging ~/.hermes/skills/
cp -r skills/web-search-strategy ~/.hermes/skills/
cp -r skills/searxng-config ~/.hermes/skills/
docker compose up -d
```

⚠️  Do NOT overwrite `~/.hermes/config.yaml`. If the user already has one,
add these lines instead of copying the template:

```yaml
web:
  search_backend: "searxng"
  extract_backend: "firecrawl"
```

To install ddgs and Scrapling (optional, for dual-backend setup):
```bash
docker compose exec hermes bash -c "uv pip install --target /opt/data/pip-packages ddgs h2 httpcore brotli fake-useragent lxml primp"
docker compose exec hermes bash -c "uv pip install --target /opt/data/pip-packages 'scrapling[all]'"
```

## Key Paths

| Path | Purpose |
|------|---------|
| `~/.hermes/config.yaml` | Hermes configuration — add `web:` section, do not replace |
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

| Service | Image | Port | Purpose |
|---------|-------|:----:|---------|
| hermes | nousresearch/hermes-agent:latest | — | Main agent (no host port by default) |
| searxng-core | searxng/searxng:latest | 8085→8080 | Search aggregation |
| searxng-valkey | valkey/valkey:9-alpine | — | SearXNG cache backend |

## Skills Included

| Skill | Location | Purpose |
|-------|----------|---------|
| hermes-web-search-debugging | skills/hermes-web-search-debugging/ | Diagnose search failures, all 8 backends |
| web-search-strategy | skills/web-search-strategy/ | Multi-round research methodology |
| searxng-config | skills/searxng-config/ | SearXNG engine config + diagnostics |

## Critical Warnings

1. **Only bing and mojeek work on self-hosted SearXNG.** brave, ddg, startpage, presearch, qwant are all permanently dead.

2. **Silent fallback**: if `search_backend: ddgs` but ddgs not installed, Hermes silently falls back to firecrawl. Verify with `get_active_search_provider().name`.

3. **ddgs not in LAZY_DEPS**: Hermes auto-installs SDKs for firecrawl/exa/parallel but NOT ddgs. After container rebuild, ddgs disappears.

4. **SEARXNG_URL must be in docker-compose.yml**: a comment in `.env` is not enough.

5. **secret_key intentionally absent**: SearXNG SHOULD auto-generate one on first startup, but if session errors occur after restart, set `SEARXNG_SECRET` in docker-compose.yml.

6. **Cloudflare blocks AI agents** (since July 2025). Use Scrapling with Playwright Chromium at `/opt/data/.playwright/` as fallback.

7. **Dual-backend pattern**: SearXNG (stable, 17-20 results) + ddgs (free, ~25% empty).
