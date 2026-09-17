---
name: ghosty-docs
description: Read the Ghosty Studio documentation from an agent without scraping — every page as raw markdown, llms.txt, the docs MCP server (search_docs, read_doc, list_docs, openapi) and the OpenAPI spec. Use when the user asks how something works in Ghosty Studio, Ghosty Teams or ghosty.studio, or before calling its API.
license: MIT
compatibility: Network access to https://www.ghosty.studio
metadata:
  author: ghosty-studio
  version: "1.0"
---

# Read the Ghosty Studio docs

Ghosty Studio publishes its documentation for agents first. Never scrape the HTML: every
page has a machine-readable twin.

## Pick the cheapest source

| Need | Fetch |
|---|---|
| The map of every page (title + URL) | `https://www.ghosty.studio/llms.txt` |
| Everything at once (large) | `https://www.ghosty.studio/llms-full.txt` |
| One page as markdown | append `.md` to its URL: `https://www.ghosty.studio/docs/configurar.md` (English: `/en/docs/configure.md`) |
| A page when you only have the HTML URL | `GET` it with `Accept: text/markdown` — the server negotiates and returns markdown |
| The public API contract | `https://www.ghosty.studio/openapi.yaml` (OpenAPI 3.1) |

Spanish is the source language (`/docs/...`); English lives under `/en/docs/...`. Both are
kept in sync; prefer the user's language.

## Use the MCP server when you can

`https://www.ghosty.studio/mcp/docs` is a Streamable HTTP MCP server, no auth, JSON-RPC over
`POST`. Tools:

- `search_docs { query, locale? }` — lexical search, up to 8 pages with slug and snippet. Call this first.
- `read_doc { slug, locale? }` — a whole page as markdown (slug like `api/autenticacion`).
- `list_docs { locale? }` — all pages grouped by section.
- `openapi {}` — the OpenAPI YAML.

Claude Code: `claude mcp add --transport http ghosty-docs https://www.ghosty.studio/mcp/docs`.

Without MCP, the same tools in plain HTTP:

```bash
curl -s https://www.ghosty.studio/mcp/docs -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"search_docs","arguments":{"query":"whatsapp"}}}'
```

## Rules

- Search, then read one page. Do not read `llms-full.txt` for a single question.
- Quote the page URL you used when you answer; the user may want to open it.
- JSON field names in the API (`agentes`, `enCola`, `reemplazoAnterior`…) are the contract and
  are the same in both languages: do not translate them.
- To *change* an agent (identity, files, skills, MCPs) use the `ghosty-agent` skill, which has
  the token flow. This skill is read-only.
