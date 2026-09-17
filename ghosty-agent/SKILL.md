---
name: ghosty-agent
description: Configure a Ghosty Studio agent (identity/system prompt, model, knowledge files in its machine, skills, custom MCP servers) through its REST API using the agent token. Use when the user asks to set up, tune, teach, or connect their Ghosty agent, or mentions ghosty.studio.
license: MIT
compatibility: Needs curl or any HTTP client and network access to https://www.ghosty.studio
metadata:
  author: ghosty-studio
  version: "1.2"
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

## Know the engine first

`GET …?fields=name,engine,model,hasMachine,prompt` before anything else. `hasMachine` decides
what applies:

| `hasMachine` | Engines | You can | Identity (`prompt`) takes effect |
|---|---|---|---|
| `true` | Ghosty · Lite, Goose | everything: files, skills, MCP, `restart`, `try` | on the next conversation, or right away with `POST …/restart` |
| `false` | Claude, Codex, DeepSeek | `GET`, `PATCH` (name, model, prompt, webSearch, channels) and `try` | on the next conversation; **never call `restart`** (it answers `409`) |

The `PATCH` response carries a `nota` saying which case you are in. Files, skills and MCP on a
machine-less engine answer `409 agente_sin_maquina`: tell the user and stop.

## What you can do

| User asks | Do |
|---|---|
| "set its identity / persona / system prompt" | write it with `references/identity.md`, `PATCH` with `{"prompt": "..."}`, then `restart` only if `hasMachine` |
| "change the model" | `GET` first (lists `models`), then `PATCH {"model": "<id>"}` (restarts by itself) |
| "give it these files / documents / knowledge" | `PUT …/files/<name>` with raw bytes, one call per file |
| "install / teach it a skill" | `PUT …/skills/<slug>` with the SKILL.md markdown (+ assets), then `POST …/restart` |
| "connect it to this MCP server" | `PUT …/mcp` with the full list of servers (it replaces; restarts by itself) |
| "what does it have?" | `GET …?full=1` → prompt, model, files, skills, mcp (`?fields=` to read just some) |
| "does it work? / test it" | `POST …/try {"text": "…"}` → the agent's answer (see Verify) |

Read `references/api.md` for exact request/response shapes before calling.

## Rules

- **Read before write.** `GET` first; `PATCH` only the fields the user asked to change.
- **Write the prompt in the user's language** and in second person ("Eres…", "You are…"), with
  the house structure in `references/identity.md` (who it is, how it talks, what it does, what
  it never does, when it asks). Keep it under ~3,000 characters: it is prepended to every
  conversation. Do not state the model: the platform injects it every turn.
- **Skills follow the Agent Skills format**: a `SKILL.md` with YAML frontmatter `name` and
  `description`, body in markdown. Slug = lowercase, digits and hyphens.
- **MCP `PUT` replaces the whole list.** `GET …/mcp` first and send back the existing servers plus
  the new one. Only `https://` URLs for HTTP servers; stdio servers need the binary to exist in the
  agent's machine (Node and Python are there).
- **Engines without their own machine** (Claude, DeepSeek, Codex) accept `GET`/`PATCH` (identity, model) only; files, skills, MCP and restart answer `409 agente_sin_maquina`. Tell the user and stop; do not retry.
- **Restart is not free**: it cuts a turn in progress. Batch changes, restart once at the end.
- Files go to the agent's working directory; tell the user the agent can `ls` them. Max 10 MB each.
- Never print the token back to the user or into logs.

## Verify

Never end on "it should work now". After the changes, `POST …/try` with a message that exercises
exactly what changed, read the answer and tell the user whether it matches:

| Changed | Ask |
|---|---|
| identity | `¿Quién eres y qué haces?` → the name and role you wrote |
| files | `¿Qué archivos tienes en tu workspace?` → lists them |
| a skill | a request the skill covers → it follows the skill's steps |
| an MCP server | one action that needs that server → it calls it |
| model | `¿Qué modelo eres?` → the label from `models` |

```bash
curl -s -X POST "$B/try" -H "Authorization: Bearer $GHOSTY_AGENT_TOKEN" \
  -H "Content-Type: application/json" -d '{"text":"¿Quién eres y qué haces?","reset":true}'
```

`reset: true` starts from a clean memory; use `session` to keep several test threads apart. A
machine that was asleep takes 5–15 s on the first call. Then tell the user in one line what
changed and what the agent answered.
