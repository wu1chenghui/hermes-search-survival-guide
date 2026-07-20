# MinerU Integration — Reference Details

This file documents setup, options, quotas, API diagnostics, and troubleshooting.
It is reference material — not loaded on every trigger. The SKILL.md covers everyday usage.

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

Flash mode is sufficient for most arXiv papers (<20 pages). Token mode is needed
for full books, theses, or when higher accuracy (vlm model) is required.

## API Endpoints

### Agent API (flash mode, no auth)

**URL mode:**
```
POST https://mineru.net/api/v1/agent/parse/url
Body: {"url": "...", "language": "en", "enable_table": true, "enable_formula": true}
→ poll GET /parse/{task_id}
→ download markdown_url
```

**File upload mode:**
```
POST https://mineru.net/api/v1/agent/parse/file
Body: {"file_name": "paper.pdf", "language": "en", ...}
→ returns task_id + signed upload URL
→ PUT file bytes to signed URL
→ poll GET /parse/{task_id}
→ download markdown_url
```

### Precision API (token required)

**Single task (URL-based):**
```
POST https://mineru.net/api/v4/extract/task
Header: Authorization: Bearer ***
Body: {"url": "...", "model_version": "vlm"}
→ poll GET /extract/task/{task_id}
→ download full_zip_url (md + json)
```

**Batch file upload (local files):**
```
Step 1: Get signed OSS upload URLs
  POST https://mineru.net/api/v4/file-urls/batch
  Header: Authorization: Bearer ***
  Body: {"files": [{"name": "paper.pdf", "data_id": "p1"}], "model_version": "vlm"}
  → response: {"data": {"batch_id": "...", "file_urls": ["https://mineru.oss-cn-shanghai.aliyuncs.com/..."]}}

Step 2: Upload files to OSS
  PUT {file_url} with file bytes
  If SSL EOF error: use requests.put(url, verify=False) + retry up to 3 times

Step 3: Poll batch results
  ⚠️ CORRECT endpoint (easy to miss in docs):
  GET https://mineru.net/api/v4/extract-results/batch/{batch_id}

  Response structure:
  {
    "data": {
      "extract_result": [{
        "data_id": "p1",
        "file_name": "paper.pdf",
        "state": "done",              ← state is nested, not at data.state
        "full_zip_url": "https://cdn-mineru.openxlab.org.cn/..."
      }]
    }
  }
```

## API Diagnostic Checklist (verified 2026-07-20)

When Precision API fails, check in this order:

| Layer | Check | Expected |
|-------|-------|----------|
| 0 Token | POST /extract/task with demo URL | code=0, returns task_id |
| 0b Token test | POST with fake token | HTTP 401 "user token expired" |
| 1 DNS | dig mineru.oss-cn-shanghai.aliyuncs.com | Resolves to 223.109.196.173 |
| 2 TCP | nc -zv IP 443 | Connection succeeded |
| 3 TLS | Python requests.head(host) | HTTP 403 (expected — no auth on GET) |
| 4 Upload | PUT to OSS signed URL | HTTP 200. If SSL EOF, use verify=False |
| 4b Poll | GET /extract-results/batch/{batch_id} | data.extract_result[0].state |
| 5 Download | GET full_zip_url | ZIP file with md + json |

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Garbled English text in output | Language defaulted to `ch` | Set `language="en"` |
| Empty output on scanned document | OCR not enabled | Set `ocr=true` |
| MCP tool not appearing after config | Hermes needs restart | `docker compose restart hermes` |
| **"task not found or expire"** | Wrong polling endpoint for batch | Use `/extract-results/batch/{batch_id}`, NOT `/extract/task/{batch_id}` |
| **State always "?" in polling** | Response structure differs from single-task | Use `data.extract_result[0].state`, NOT `data.state` |
| **OSS upload SSL EOF** | Alibaba Cloud OSS SSL quirk from this network | `requests.put(url, verify=False)` + retry up to 3 times |
| **"failed to read file" after upload** | OSS signed URL not readable by MinerU backend | Use Agent API file upload instead, or host file on public URL |
| Batch submit returns code=0 but no data | Token invalid or expired | Check with Layer 0 test |

## Official Resources

- API docs: https://mineru.net/apiManage/docs
- Ecosystem: https://mineru.net/ecosystem
- GitHub: https://github.com/opendatalab/MinerU (75k stars)
- Ecosystem repo: https://github.com/opendatalab/MinerU-Ecosystem
- MCP server: https://pypi.org/project/mineru-open-mcp/
