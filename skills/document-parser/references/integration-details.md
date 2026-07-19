# MinerU Integration — Reference Details

This file documents setup, options, quotas, and troubleshooting. It is
reference material — not loaded on every trigger. The SKILL.md covers
everyday usage.

---

## Integration: How MinerU is Connected

We use the **local MCP server via uvx** (stdlib transport):

```yaml
# ~/.hermes/config.yaml
mcp_servers:
  mineru:
    command: uvx
    args:
    - mineru-open-mcp
    env:
      MINERU_API_TOKEN: "<token>"
```

The MCP server (`mineru-open-mcp` on PyPI) is launched by Hermes as a subprocess.
It exposes the `parse_documents` tool. Both flash mode (no token) and token mode
work through the same tool — if `MINERU_API_TOKEN` is set, token mode is used;
otherwise flash mode with lower limits.

**Why this over alternatives:**

| Option | Why not chosen |
|--------|---------------|
| Remote MCP (`mcp.mineru.net/mcp`) | Only token mode, no flash. Depends on external service uptime. |
| CLI (`mineru-open-api`) | Requires npm/go install. Output parsing is fragile. |
| Direct API (Python) | Requires writing API code every time. No native tool integration. |

## Token Management

Token stored at `/opt/data/.mineru-token`. Created at https://mineru.net/apiManage.

**NEVER paste token into conversation.** GitHub secret scanning auto-revokes
tokens that appear in chat. Always read from file in `execute_code`:

```python
with open("/opt/data/.mineru-token") as f:
    token = f.read().strip()
```

The MCP server reads the token from the `MINERU_API_TOKEN` environment variable
set in `~/.hermes/config.yaml`. The token never enters the conversation.

## Free Tier Quotas

| | Flash mode | Token mode |
|---|---|---|
| Auth | None (IP rate-limited) | Bearer token |
| File size | ≤10MB | ≤200MB |
| Pages | ≤20 | ≤200 |
| Daily quota | Rate-limited | 1 000 pages high priority |
| Output | Markdown only | md, html, latex, docx, json |
| Model | pipeline | vlm (default) or pipeline |
| Formula/Table/OCR | ✅ | ✅ |
| Batch | ❌ | ✅ (≤200 files) |

Flash mode is sufficient for most arXiv papers (<20 pages). Token mode is
needed for full books, theses, or when higher accuracy (vlm model) is required.

## API Endpoints (for reference)

These are the underlying APIs the MCP server calls. Direct use is only needed
for debugging or if the MCP server is unavailable.

### Agent API (flash mode, no auth)

```
POST https://mineru.net/api/v1/agent/parse/url
Body: {"url": "...", "language": "en", "enable_table": true, "enable_formula": true}
→ poll GET /parse/{task_id} → download markdown_url
```

### Precision API (token required)

```
POST https://mineru.net/api/v4/extract/task
Header: Authorization: Bearer <token>
Body: {"url": "...", "model_version": "vlm"}
→ poll GET /extract/task/{task_id} → download full_zip_url (md + json)
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Garbled English text in output | Language defaulted to `ch` | Set `language="en"` |
| Empty output on scanned document | OCR not enabled | Set `ocr=true` |
| MCP tool not appearing after config | Hermes needs restart | `docker compose restart hermes` |
| Tool exists but returns errors | Token expired or MCP server issue | Check token at mineru.net, try flash mode (remove token env) |
| "File too large" error | Exceeded flash mode limits (10MB/20p) | Use token mode or switch to Precision API directly |

## Official Resources

- API docs: https://mineru.net/apiManage/docs
- Ecosystem: https://mineru.net/ecosystem
- GitHub: https://github.com/opendatalab/MinerU (75k stars)
- Ecosystem repo: https://github.com/opendatalab/MinerU-Ecosystem
- MCP server: https://pypi.org/project/mineru-open-mcp/
