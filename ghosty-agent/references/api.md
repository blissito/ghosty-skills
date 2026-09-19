# Ghosty Studio — agent configuration API

Base: `https://www.ghosty.studio/api/v2/agents/{id}` · Auth: `Authorization: Bearer gat_…`
(agent token) or an owner OAuth2 bearer with `agents:write`. JSON unless noted.
Full spec: https://www.ghosty.studio/openapi.yaml · Docs: https://www.ghosty.studio/docs/configurar

## GET /
Returns `{ id, name, engine, hasMachine, model, models: [{id,label}], prompt, channels, webSearch, mcp, messengerPages: [{pageId,pageName}], starters, tools: { gs: {name: bool}, extensions: {name: bool} } }`.
`hasMachine` (bool) says whether files/skills/MCP/restart exist for this engine.
Add `?full=1` to also get `files: [{path,size}]`, `skills: [{slug,description,files}]` and `extensions: [{name,enabled}]`
(wakes the machine if asleep). Add `?fields=prompt,model` to get only those keys (`id` always).

```bash
curl -s "$B" -H "Authorization: Bearer $GHOSTY_AGENT_TOKEN"
```

## PATCH /
Body: any of `{ "name", "model", "prompt", "webSearch": bool, "channels": { "teams": bool, "web": bool, "whatsapp": bool, "messenger": bool }, "starters": ["…up to 5, ≤80 chars"], "tools": { "gs": { "<name>": bool }, "extensions": { "<name>": bool } } }`.
`tools.gs` today: `programar_seguimiento` (schedule a future turn) and `actualizar_identidad` (the agent rewrites or appends to its own prompt — it only works when the owner talks to it from Studio or the Mac app; from Messenger/WhatsApp/Teams the call is refused).
`tools.canales` decides per channel: `{ "canales": { "messenger": { "actualizar_identidad": false } } }` (channels: `chat` = Studio/Mac app, `teams`, `whatsapp`, `messenger`, `programado`, `prueba`; the GET returns the effective matrix). By default `actualizar_identidad` is only on in `chat`.
`tools` turns tools off (`false`) or back on (`true`) by name and merges with what is saved. `tools.gs` are the tools gs lends the agent (the GET lists them all with their state); `tools.extensions` are the machine's `config.yaml` extensions (`developer` = shell + files, `todo`, `analyze`, `easybits`…; `?full=1` returns the real list as `extensions`) and changing them restarts the machine. The veto is enforced: an off tool disappears from the list and is refused when called.
Response: the same as GET plus `aplicado: ["set-prompt", …]` and, after a `prompt` change, `nota`
telling whether a `restart` applies (machine) or the identity simply enters on the next
conversation (no machine). `model` restarts the agent.

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

## GET /whatsapp  ·  PUT /whatsapp
Which WhatsApp numbers this agent answers. Numbers belong to the workspace and are paired from Sales
(`salesUrl`); here you only choose who answers. `PUT {"numbers": ["<id>", …]}` leaves the agent
answering exactly those (one number → one agent; unchecking mutes it, the pairing stays).

```bash
curl -s "$B/whatsapp" -H "Authorization: Bearer $GHOSTY_AGENT_TOKEN"
```
→ `{ numbers: [{id, label, mine, takenByOther}], hasSales, salesUrl }`

## GET /bundle  ·  POST /bundle
The whole agent as an Eve-style directory (`instructions.md`, `skills/<slug>/SKILL.md`, `knowledge/…`, `mcp.json`),
as JSON: `{ "layout": "eve", "files": [{ "path", "contentBase64" }] }`. `POST` takes the same shape and imports
it: `instructions.md` → prompt, `skills/x.md` or `skills/x/SKILL.md` → skills (a frontmatter is added when missing),
`knowledge/…` → files. Eve's `tools/`, `channels/`, `schedules/` are not executed and come back in `ignorado`.

```bash
curl -s "$B/bundle" -H "Authorization: Bearer $GHOSTY_AGENT_TOKEN" > agent.json
```

## POST /restart
→ `{ reiniciado: true }`. Cuts a running turn; disk survives. Only with `hasMachine: true`;
otherwise `409 agente_sin_maquina`.

## POST /try
Body: `{ "text": "…", "session"?: "a-z0-9_-", "reset"?: bool }`. One full turn to text, no stream,
up to 180 s. `session` (default `default`) keeps separate memories; `reset: true` forgets that
session first (with no `text` it only forgets). One turn at a time per session (`409 turno_en_curso`).
Works on every engine; consumes balance like any turn.

```bash
curl -s -X POST "$B/try" -H "Authorization: Bearer $GHOSTY_AGENT_TOKEN" \
  -H "Content-Type: application/json" -d '{"text":"¿Quién eres?","reset":true}'
```
→ `{ "text": "Soy Ghosty…", "error": null, "session": "default" }` · `502` if the turn failed with no text.

## Errors
`400` invalid body (message in `error`) · `404` unknown id/token · `405` wrong method ·
`409 agente_sin_maquina` only `POST /restart` needs an engine with its own machine (Ghosty · Lite or Goose). Files, skills, MCP, `GET`/`PATCH` work on every engine: the agent's **bundle** (`instructions.md`, `skills/`, `knowledge/`) lives in Studio (`storage: "bundle"`) and is seeded into the worker's cwd on each turn when it changed; on machines (`storage: "box"`) it is also written to `/data/agent` and `/data/work` ·
`413` too big · `502` saved but the machine did not take it (retry `POST /restart`).
