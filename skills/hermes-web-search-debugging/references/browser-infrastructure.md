# Hermes Browser Infrastructure

Discovered 2026-07-17. Hermes bundles its own headless Chromium — no
external browser install needed.

## Built-in stack

```
browser_navigate(url)
       ↓
/opt/hermes/node_modules/agent-browser/bin/agent-browser-linux-x64   (Rust CLI)
       ↓
/opt/data/.agent-browser/browsers/chrome-149.0.7827.54/chrome        (Chrome for Testing)
       ↓
--headless=new --no-sandbox --disable-dev-shm-usage
       ↓
accessibility tree snapshot → LLM
```

## Key paths

| Component | Path |
|-----------|------|
| agent-browser binary | `/opt/hermes/node_modules/agent-browser/bin/agent-browser-linux-x64` |
| Chrome binary | `/opt/data/.agent-browser/browsers/chrome-149.0.7827.54/chrome` |
| Chrome user data | `/tmp/agent-browser-chrome-<uuid>/` |

## What this means

- `browser_navigate` works immediately — no setup needed
- Chrome 149 runs headless with `--no-sandbox` (Docker-required)
- This is separate from Scrapling's Playwright Chromium (which lives under
  `/opt/hermes/.playwright/` and failed to install due to directory permissions)
- For Cloudflare bypass, Scrapling's `StealthyFetcher` needs Playwright's own
  Chromium with stealth patches. Without it (`scrapling install` failed),
  fall back to detection+skip (check page title for "Just a moment...")
