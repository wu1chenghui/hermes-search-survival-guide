---
name: document-parser
description: >-
  Parse any document (PDF, DOCX, PPTX, XLSX, PNG, JPG, HTML, web pages)
  into LLM-readable Markdown via MinerU. Use when the user asks to read,
  analyze, extract, or convert content from a document — regardless of
  format. Supports 80+ languages, table/formula/OCR recognition.
version: 1.0.0
---

# Document Parser — MinerU Integration

## When to Use

- User provides a file path or URL, and asks to read/analyze/extract its content
- Common triggers: "read this", "analyze this document", "extract from", "parse this", "what's in this file"
- Covers ALL formats MinerU supports: PDF, DOCX/DOC, PPTX/PPT, XLSX/XLS, PNG, JPG, JPEG, JP2, WEBP, GIF, BMP, HTML, plus any URL

## The Tool

`mcp_mineru_parse_documents` — converts documents into clean Markdown with:
- LaTeX for mathematical formulas
- HTML for tables
- OCR for scanned documents and images (80+ languages)
- Reading-order preserved for complex layouts

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `source` | string | required | File path or URL |
| `pages` | string | all pages | Page range, e.g. "1-5" or "1,3,5-7". PDF only. |
| `language` | string | "ch" | OCR/baseline language: en, ch, japan, korean, latin, arabic, etc. |
| `model` | string | "vlm" | vlm (high accuracy) or pipeline (zero hallucination) |
| `ocr` | boolean | false | Enable OCR for scanned documents and images |
| `formula` | boolean | true | Enable formula recognition |
| `table` | boolean | true | Enable table recognition |

## Usage Examples

```
"Summarize this paper: https://arxiv.org/pdf/..."
→ parse_documents(source=url, language="en")

"What's in this scanned receipt?" (shows a JPG)
→ parse_documents(source="receipt.jpg", ocr=true)

"Extract the tables from this Excel file"
→ parse_documents(source="data.xlsx", table=true)

"Read pages 5-8 of this PDF"
→ parse_documents(source="book.pdf", pages="5-8", language="en")

"Parse this Chinese Word document"
→ parse_documents(source="报告.docx", language="ch")
```

## Critical Pitfalls

1. **Language defaults to Chinese.** For English papers, always set `language="en"`. Forgetting this causes OCR to misinterpret English text as Chinese characters.
2. **For scanned documents and images, set `ocr=true`.** Without it, MinerU skips OCR entirely and returns empty or garbled output.
3. **Token is at `/opt/data/.mineru-token`** and configured in `~/.hermes/config.yaml` under `mcp_servers.mineru.env.MINERU_API_TOKEN`. Flash mode works without token but has lower limits (10MB/20 pages).

## When to Load Reference Details

The MCP tool auto-selects mode: if `MINERU_API_TOKEN` is set (it is — configured in
`~/.hermes/config.yaml`), it uses Precision (vlm, higher accuracy, up to 200MB/200
pages). Without token, it falls back to flash mode.

**Prefer Precision by default** — our token gives 1000 pages/day high priority, and
the accuracy difference matters for math-heavy papers. Only fall back to flash if:

- Token call fails (rare) → remove `MINERU_API_TOKEN` env to force flash
- You want instant results without API overhead → flash is faster (3-5s vs 8-12s)

**Load `references/integration-details.md` whenever:**
- Precision API returns errors → use the 5-layer diagnostic checklist
- You need to call the API directly (bypass MCP) for batch uploads or debugging
- You need to check quotas, endpoint details, or SSL workarounds

To load: `skill_view(name="document-parser", file_path="references/integration-details.md")`
For setup details, API endpoints, free tier quotas, and the 5-layer API diagnostic checklist, see `references/integration-details.md`.
