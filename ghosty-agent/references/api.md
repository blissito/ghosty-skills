# Ghosty Studio — agent configuration API

Base: `https://www.ghosty.studio/api/v2/agents/{id}` · Auth: `Authorization: Bearer gat_…`
(agent token) or an owner OAuth2 bearer with `agents:write`. JSON unless noted.
Full spec: https://www.ghosty.studio/openapi.yaml · Docs: https://www.ghosty.studio/docs/configurar

## GET /
Returns `{ id, name, engine, model, models: [{id,label}], prompt, channels, webSearch, mcp }`.
Add `?full=1` to also get `files: [{path,size}]` and `skills: [{slug,description,files}]`
(wakes the machine if asleep).

```bash
curl -s "$B" -H "Authorization: Bearer $GHOSTY_AGENT_TOKEN"
```

## PATCH /
Body: any of `{ "name", "model", "prompt", "webSearch": bool, "channels": { "teams": bool } }`.
Response: the same as GET plus `aplicado: ["set-prompt", …]`. `model` restarts the agent.

```bash
curl -s -X PATCH "$B" -H "Authorization: Bearer $GHOSTY_AGENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"prompt":"Eres Nora, asistente de la clínica Dental Sur. Contestas en español, corto, y nunca das diagnósticos."}'
```

## PUT /files/{path}  ·  DELETE /files/{path}  ·  GET /files[/dir]
Raw bytes in the body (no multipart, no base64). Max 10 MB. `path` is relative, no `..`.

```bash
curl -s -X PUT "$B/files/precios-2026.pdf" -H "Authorization: Bearer $GHOSTY_AGENT_TOKEN" \
  -H "Content-Type: application/pdf" --data-binary @precios-2026.pdf
```
→ `{ path, bytes, en: "/data/work/precios-2026.pdf" }`

## PUT /skills/{slug}  ·  DELETE /skills/{slug}  ·  GET /skills
Body: `{ "markdown": "<SKILL.md content>", "assets": [{ "name": "scripts/x.py", "contentBase64": "…" }] }`.
Max 25 MB total. Then `POST /restart` so the agent loads it.

```bash
jq -n --rawfile md SKILL.md '{markdown:$md}' | \
curl -s -X PUT "$B/skills/cotizaciones" -H "Authorization: Bearer $GHOSTY_AGENT_TOKEN" \
  -H "Content-Type: application/json" -d @-
```

## GET /mcp  ·  PUT /mcp
Body: `{ "servers": [ …full list… ] }`. Each server is one of:
- `{ "name": "notion", "type": "http", "url": "https://mcp.notion.com/mcp", "headers": { "Authorization": "Bearer …" } }`
- `{ "name": "fs", "command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "/data/work"], "env": {} }`

Names: `a-z 0-9 - _`, max 20 servers. The agent restarts automatically.

## POST /restart
→ `{ reiniciado: true }`. Cuts a running turn; disk survives.

## Errors
`400` invalid body (message in `error`) · `404` unknown id/token · `405` wrong method ·
`409 agente_sin_maquina` files, skills, MCP and restart need an engine with its own machine (Ghosty · Lite or Goose); `GET`/`PATCH` work on every engine ·
`413` too big · `502` saved but the machine did not take it (retry `POST /restart`).
