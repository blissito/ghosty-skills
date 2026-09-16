---
name: ghosty-agent
description: Configure a Ghosty Studio agent (identity/system prompt, model, knowledge files in its machine, skills, custom MCP servers) through its REST API using the agent token. Use when the user asks to set up, tune, teach, or connect their Ghosty agent, or mentions ghosty.studio.
---

# Configure a Ghosty Studio agent

A Ghosty Studio agent runs on its own isolated machine (disk, terminal, memory). You configure it
over HTTPS with **its own token**; nothing runs on the user's computer.

## Setup (once)

1. Ask the user for the agent id and token. Both are in Ghosty Studio → **Agentes → their agent →
   Conexión con tu editor → Generar token** (token looks like `gat_…`; id is in the page URL
   `/app/agents/<id>`).
2. Keep them in env vars, never in command arguments or committed files:

```bash
export GHOSTY_AGENT_ID="<id>"
export GHOSTY_AGENT_TOKEN="gat_…"
```

Base URL: `https://www.ghosty.studio/api/v2/agents/$GHOSTY_AGENT_ID`. Every call:
`-H "Authorization: Bearer $GHOSTY_AGENT_TOKEN"`. Wrong token or id → `404` (do not retry).

## What you can do

| User asks | Do |
|---|---|
| "set its identity / persona / system prompt" | `PATCH` with `{"prompt": "..."}` then `POST …/restart` |
| "change the model" | `GET` first (lists `models`), then `PATCH {"model": "<id>"}` (restarts by itself) |
| "give it these files / documents / knowledge" | `PUT …/files/<name>` with raw bytes, one call per file |
| "install / teach it a skill" | `PUT …/skills/<slug>` with the SKILL.md markdown (+ assets), then `POST …/restart` |
| "connect it to this MCP server" | `PUT …/mcp` with the full list of servers (it replaces; restarts by itself) |
| "what does it have?" | `GET …?full=1` → prompt, model, files, skills, mcp |

Read `references/api.md` for exact request/response shapes before calling.

## Rules

- **Read before write.** `GET` first; `PATCH` only the fields the user asked to change.
- **Write the prompt in the user's language** and in second person ("Eres…", "You are…"). Keep it
  under ~3,000 characters: it is prepended to every conversation.
- **Skills follow the Agent Skills format**: a `SKILL.md` with YAML frontmatter `name` and
  `description`, body in markdown. Slug = lowercase, digits and hyphens.
- **MCP `PUT` replaces the whole list.** `GET …/mcp` first and send back the existing servers plus
  the new one. Only `https://` URLs for HTTP servers; stdio servers need the binary to exist in the
  agent's machine (Node and Python are there).
- **Restart is not free**: it cuts a turn in progress. Batch changes, restart once at the end.
- Files go to the agent's working directory; tell the user the agent can `ls` them. Max 10 MB each.
- Never print the token back to the user or into logs.

## After changes

Tell the user in one line what changed and suggest a test message for the agent, e.g.
"Ask it: *¿qué archivos tienes en tu workspace?*".
