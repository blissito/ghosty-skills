---
name: ghosty-agent
description: Configure, test and chat with a Ghosty Studio agent (identity/system prompt, model, knowledge files in its machine, skills, custom MCP servers) with the `ghosty` CLI, or its REST API as a fallback. Use when the user asks to set up, tune, teach, test or connect their Ghosty agent, or mentions ghosty.studio.
license: MIT
compatibility: Needs Node 22+ (for npx @ghostystudio/cli) or curl, and network access to https://www.ghosty.studio
metadata:
  author: ghosty-studio
  version: "1.19"
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

**Remote box, SSH or CI?** `login` switches by itself to a device code (`--device` forces it): the
`login_url` line then carries `"code"` too; pass the link to the user and tell them to check the
code matches before authorizing.

**Several accounts** (the user's and a customer's)? `--profile <name>` (or `GHOSTY_PROFILE`) keeps
a separate saved session per name: `ghosty --profile cliente login --json`, then the same
`--profile` on every command.

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

`ghosty agents get <id> --fields name,engine,model,needsRestart,prompt --json` before anything else. Every agent
has its own machine and disk, so files, skills, MCP and `try` work on all of them. `needsRestart` only
says how new config gets in:

| `needsRestart` | Engines | Files, skills and MCP take effect | Identity (`prompt`) takes effect |
|---|---|---|---|
| `true` | Ghosty · Lite, Goose (ACP) | skills after `ghosty agents restart <id>`; MCP restarts by itself; files right away | on the next conversation, or right away with `restart` |
| `false` | Claude, Codex, DeepSeek, Gemini (pool) | on the next turn, by themselves | on the next turn; **never call `restart`** (it answers `409 restart_no_aplica`) |

The `PATCH` and skill responses carry a `nota` saying which case you are in. (`hasMachine` is
legacy and always `true`.)

## What you can do

| User asks | Do |
|---|---|
| "create a new agent" | `ghosty agents create --name <name> [--engine <engine>] [--model <model-id>] [--prompt-file PROMPT.md] [--env K=V,…] [--workspace <slug>] [--channels teams=off] --json` → `id`. A model outside the engine answers 400 with the valid list. `--workspace` = born in that workspace, owned by its owner, active in its Teams |
| "set its identity / persona / system prompt" | write it with `references/identity.md` to a file, `ghosty agents set <id> --prompt-file PROMPT.md`, then `ghosty agents restart <id>` only if `needsRestart` |
| "let it see my Drive / use my connector" | `ghosty agents set <id> --connect google-drive` (owner only; the account must have it connected first). Files are the ones the owner picked in Conectores → Google Drive |
| "edit its (long) prompt" | `ghosty agents get <id> --prompt-out PROMPT.md`, edit the file, `ghosty agents set <id> --prompt-file PROMPT.md` |
| "base its prompt on another agent's (a branch, a variant)" | write an overlay `{"edits":[{"what","find","replace","regex"?,"all"?}]}`, `ghosty agents set <id> --prompt-base <base-id> --prompt-overlay overlay.json --dry-run --out /tmp/p.md`, check, then without `--dry-run`. It regenerates when the base changes; its own prompt can't be edited while linked (`--prompt-base none` unlinks) |
| "give it the CRM / sales tools" | `ghosty agents set <id> --toolsets +crm` (a plain list replaces; `-x` removes) |
| "share it with / give access to <email>" | `ghosty agents share <id> <email> [--role editor\|admin]` (keeps ownership); `ghosty agents shares <id>` lists; `unshare` needs `--yes` |
| "turn off / on this skill" | `ghosty skills off\|on <id> <slug>` (its own or one of Ghosty's; `skills ls` shows `on`) |
| "delete this agent" | confirm with the user first, then `ghosty agents rm <id>` (owner only; `409` names the workspace where it is active) |
| "switch it to another engine" | `ghosty agents set <id> --engine <engine> [--model <model-id>]` (the model belongs to the NEW engine) |
| "change the model" | `ghosty agents get <id> --json` (lists `models`), then `ghosty agents set <id> --model <model-id>` (restarts by itself) |
| "make it think more / less" | Ghosty · Lite: `ghosty agents set <id> --env GHOSTY_THINKING_EFFORT=off\|low\|medium\|high\|max`. Codex: `--env FLEET_EFFORT=none\|minimal\|low\|medium\|high\|xhigh\|max` |
| "give it these files / documents / knowledge" | `ghosty files put <id> <name> --file <local>`, one per file |
| "install / teach it a skill" | `ghosty skills add <id> <slug> --file SKILL.md` (or just `<slug>` from the catalog: `ghosty skills ls <id>`), then `ghosty agents restart <id>` only if `needsRestart` |
| "connect it to this MCP server" | `ghosty mcp get <id> --json > servers.json`, add the server, `ghosty mcp set <id> --file servers.json` (replaces; restarts by itself) |
| "what does it have?" | `ghosty agents get <id> --json` → prompt, model, files, skills, mcp |
| "does it work? / test it" | `ghosty try <id> "…" --json` → the agent's answer (see Verify) |
| "talk to it / ask it something" | `ghosty chat <id> "…" --json` → streams `chunk` lines, ends with `done` |
| "test it as a WhatsApp customer / with a photo / with earlier context" | `ghosty try <id> "…" --waba --session <made-up phone> [--media FILE] [--history FILE --reset] --json` → `sent[]` (what it would send, tools included: `kind` text/voice/file/rich with `rich` ubicacion/cita/boton/contacto/reaccion; nothing leaves). A customer attachment exactly as Formmy delivers it: add `--as-formmy` → `media.copied` |
| "show me this customer's chat on the board / did the attachment arrive?" | `ghosty board ls <id> [--q name-or-phone] --json`, then `ghosty board show <id> <folio\|phone> [--limit 50] --json` → `messages[]` with `role`, `content`, `mediaType`, `mediaCopied` |
| "replay these real conversations" | `ghosty try <id> --replay sample.json --out ./replay [--max N]`; read `./replay/replay.json`; `--resume` if it was cut |
| "is it configured right? / why does it answer badly?" | `ghosty agents doctor <id> --json` → `checks[]` with `level` (ok/warn/error) and a `fix` command each; exit 1 = something to fix |
| "why did it fail? / it didn't answer" | `ghosty turns ls <id> --errors --since 24h --json` → failed turns with their `error` |
| "what did it answer to that? / audit a reply" | `ghosty turns show <id> [turn-id] --json` → `input` (what came in) and `output` (what it answered), any engine or channel |
| "why was it slow? / where did the time go?" | `ghosty turns show <id> [turn-id] --json` → `byTool` (seconds per tool), `steps` (timeline), `totals.outputTokens`; pool engines only (`timeline: false` otherwise) |
| "is it running the new image? / it still behaves old" | `ghosty agents box <id> --check <path-the-new-image-brings> --json` → `stale`; if > 0, `ghosty agents box <id> --recycle`, one turn, check again (ACP: `agents restart`) |
| "which PDF templates does it have?" | `ghosty skills templates <id> --json` → `templates[]` with `source` (`agente:<skill>` or `casa`); its own go in `<skill>/pdf-templates/<name>.html` |
| "give it this database / what can it write" | `ghosty dbs ls <id>`, `ghosty dbs tables <id> <db>`, `ghosty dbs grant <id> <db> [--external-write t1,t2] [--external-rows t:col+col]` (tables it may write from WhatsApp/Messenger; tables where each customer sees only their own rows, matched by phone column — use it for any customers/orders table) |
| "copy its data from EasyBits" | `ghosty dbs import <id> <db> --from easybits:<db-id> --dry-run` first (needs `EASYBITS_API_KEY`), show the plan, then run without `--dry-run` (`--yes` if replacing rows) |
| "let it read / send this document" | `ghosty agents docs ls <id>`, `ghosty agents docs grant <id> "<name>" [--write] [--no-deliver]`; a new one: `ghosty agents docs add <id> --url <url> \| --file <path> [--description T]` (uploads and grants) |
| "answer with voice notes" | `ghosty agents set <id> --voice elevenlabs:<voice-id>\|kokoro:em_santa --voice-replies auto` |
| "answer in this WhatsApp group / make the group team or customer" | personal WhatsApp only: `ghosty channels wa groups ls <id> --json`, then confirm with the user and `ghosty channels wa groups set <id> "<group>" --role equipo\|cliente --on --yes` (`--dry-run` first) |
| "move / migrate this WhatsApp number from EasyBits or Formmy" | Business: `ghosty channels whatsapp link <id> --integration <formmy-id> --answer on\|off --dry-run --json` → show `plan`; confirm with the user (on = answers real customers), then without `--dry-run` and `--yes`; back: `unlink`. Its recent cards/messages/orders: `ghosty channels whatsapp history <id> --integration <formmy-id> --days 7 --dry-run`, then without `--dry-run` (`--yes`; re-running never duplicates). Personal: `ghosty channels wa import <id> --from-file F` (EasyBits JSON) or `--from-dir DIR` (Baileys multi-file folder) `--own-number yes\|no --dry-run` (the user turns off EasyBits WITHOUT logout first), then `wa status` |
| "answer this WhatsApp number" | confirm with the user (real customers), then `ghosty channels whatsapp enable <id> <number> --yes`; `--dry-run` shows the plan |
| "save my ElevenLabs / MercadoPago key" | never put the key in a command: ask the user to run `ghosty credentials set <provider>` themselves (hidden prompt), or use `--from-env VAR` |
| "remove this skill" | `ghosty skills rm <id> <slug> --yes` keeps a local copy and returns `restore`; `ghosty skills get <id> <slug> --out DIR` to back one up first (without `--out` it only lists) |
| "how much have I used? / am I out of usage?" | `ghosty usage --json` → `plan.name`, `week.pct` / `month.pct` (0–1), `exhausted` |
| "let people try it without an account / share a demo" | `ghosty agents demo <id> --slug <name> --on [--vence "YYYY-MM-DDTHH:MM"] [--welcome T] [--starter T] [--chips "a\|b"] --json` → `url`; no flags = status; `--rotate` if the link leaked, `--off` to stop |
| "what files are in my account?" | `ghosty me files ls [--kind document] --json`; `me files upload <path>`, `me files rm <file-id>` |
| "clean up the board / archive test cards" | `ghosty board archive <id> --integration X \| --column X \| --before 30d --dry-run` first, then with `--yes` after the user confirms (reversible) |
| "which conversations does it have?" | `ghosty conversations ls <id> --json`; read one with `ghosty conversations show <id> <conv-id> --json` (`messages[]` with `role`, `text`); continue one with `ghosty chat <id> "…" --conversation <conv-id>` |
| "what's in its database? / fix this row" | `ghosty dbs query <id> <db> "SELECT …" [--arg V]… --json` (read only); `--write` only for a change the user asked for |
| "stop giving it this database" | `ghosty dbs revoke <id> <db>` (prints the `grant` that undoes it) |
| "give an agent to each of MY customers (partner)" | not for a normal owner: see https://www.ghosty.studio/docs/cli/partners.md (`ghosty partner --help`) |
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
- **Pool engines** (Claude, Codex, DeepSeek, Gemini) have their own machine too: files, skills and MCP work and enter on the next turn. Only `restart` answers `409 restart_no_aplica` there; skip it.
- **Restart is not free**: it cuts a turn in progress. Batch changes, restart once at the end.
- Files go to the agent's working directory; tell the user the agent can `ls` them. Max 10 MB each.
- Never print the token back to the user or into logs.
- **Destructive commands need `--yes`** when you run them (no terminal): `agents rm`, `files rm`, `skills rm`, `credentials rm`, `board archive`, `dbs import` over existing rows, `channels whatsapp enable`, `channels wa groups set --on`, `channels whatsapp link|unlink|history`, `channels wa import`, `agents unshare`. Confirm with the user BEFORE adding it; prefer `--dry-run` first where it exists.

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
