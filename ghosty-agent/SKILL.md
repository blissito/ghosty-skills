---
name: ghosty-agent
description: Configure, test and chat with a Ghosty Studio agent (identity/system prompt, model, knowledge files in its machine, skills, custom MCP servers) with the `ghosty` CLI, or its REST API as a fallback. Use when the user asks to set up, tune, teach, test or connect their Ghosty agent, or mentions ghosty.studio.
license: MIT
compatibility: Needs Node 22+ (for npx @ghostystudio/cli) or curl, and network access to https://www.ghosty.studio
metadata:
  author: ghosty-studio
  version: "1.4"
---

# Configure a Ghosty Studio agent

A Ghosty Studio agent runs on its own isolated machine (disk, terminal, memory). You configure it
over HTTPS with **its own token**; nothing runs on the user's computer.

## Setup (once): use the CLI

Run it with `npx -y @ghostystudio/cli <command>` (or `ghosty <command>` if installed globally).
Always pass `--json` and read stdout as JSON; notices go to stderr.

**Sign in (the user just opens a link):**

```bash
npx -y @ghostystudio/cli login --json
# first line: {"event":"login_url","url":"…"}  → show this link to the user and ask them to open it
# then:       {"event":"logged_in","email":"…"} → done; the session is saved and renews itself
```

Run it in the background or with a long timeout: it waits (up to 5 min) for the user to sign in
in their browser. Any command exiting with code **3** means "not signed in" → run `login` again.

Then `ghosty agents ls --json` gives the agent ids.

**No browser available** (CI, remote box)? Ask the user for the agent token (Ghosty Studio →
**Agentes → their agent → Conexión con tu editor → Generar token**, looks like `gat_…`) and put it
in the environment, never in arguments or committed files: `export GHOSTY_TOKEN="gat_…"`. It
reaches only that agent (no `agents ls`, no `chat`).

Exit codes: `0` ok · `1` API error (read the message, don't retry blindly) · `2` usage error
(check `--help`) · `3` not signed in.

### Fallback: raw HTTP

If Node is not available, use curl with the agent token. Base URL:
`https://www.ghosty.studio/api/v2/agents/$GHOSTY_AGENT_ID`, header
`Authorization: Bearer $GHOSTY_AGENT_TOKEN`. Wrong token or id → `404` (do not retry). Shapes in
`references/api.md`.

**Agent hosted on EasyBits** (`ghosty-lite` / `goose` template)? Same contract for `/prompt`,
`/files`, `/skills/{slug}`, `/mcp`, `/restart`: base `https://www.easybits.cloud/api/v2/agents/$AGENT_ID`
with `Authorization: Bearer $EASYBITS_API_KEY` (the owner's key, not an agent token). Identity
there is `PATCH /` with `{ systemPrompt, systemPromptMode }` (`replace` = only your prompt).

## Know the engine first

`ghosty agents get <id> --fields name,engine,model,hasMachine,prompt --json` before anything else. `hasMachine` decides
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
| "set its identity / persona / system prompt" | write it with `references/identity.md` to a file, `ghosty agents set <id> --prompt-file PROMPT.md`, then `ghosty agents restart <id>` only if `hasMachine` |
| "change the model" | `ghosty agents get <id> --json` (lists `models`), then `ghosty agents set <id> --model <model-id>` (restarts by itself) |
| "give it these files / documents / knowledge" | `ghosty files put <id> <name> --file <local>`, one per file |
| "install / teach it a skill" | `ghosty skills add <id> <slug> --file SKILL.md` (or just `<slug>` from the catalog: `ghosty skills ls <id>`), then `ghosty agents restart <id>` |
| "connect it to this MCP server" | `ghosty mcp get <id> --json > servers.json`, add the server, `ghosty mcp set <id> --file servers.json` (replaces; restarts by itself) |
| "what does it have?" | `ghosty agents get <id> --json` → prompt, model, files, skills, mcp |
| "does it work? / test it" | `ghosty try <id> "…" --json` → the agent's answer (see Verify) |
| "talk to it / ask it something" | `ghosty chat <id> "…" --json` → streams `chunk` lines, ends with `done` |
| "have it do X every day / at 9 / remind me" | `ghosty schedule add <id> "…" --at ISO \| --in 30m [--every 12h --until ISO] [--title T]`; each run notifies the user's phone, an answer of exactly `OK` stays silent. `ghosty schedule ls|rm <id>` |

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
npx -y @ghostystudio/cli try <id> --reset --json
npx -y @ghostystudio/cli try <id> "¿Quién eres y qué haces?" --json
```

`--reset` starts from a clean memory; use `--session <name>` to keep several test threads apart. A
machine that was asleep takes 5–15 s on the first call. Then tell the user in one line what
changed and what the agent answered.
