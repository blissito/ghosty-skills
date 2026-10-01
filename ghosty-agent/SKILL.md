---
name: ghosty-agent
description: Configure, test and chat with a Ghosty Studio agent (identity/system prompt, model, knowledge files in its machine, skills, custom MCP servers) with the `ghosty` CLI, or its REST API as a fallback. Use when the user asks to set up, tune, teach, test or connect their Ghosty agent, or mentions ghosty.studio.
license: MIT
compatibility: Needs Node 22+ (for npx @ghostystudio/cli) or curl, and network access to https://www.ghosty.studio
metadata:
  author: ghosty-studio
  version: "1.42"
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
| the user reads Spanish / English | the CLI follows the locale; force it with `--lang es` or `--lang en` (JSON keys never change) |
| any command that takes `<id>` | an agent's name works too (`ghosty agents get pia-0`); if the CLI says several share the name, use the id it lists. `ghosty use <workspace>` sets a default workspace (boards, create, ls) |
| "create a new agent" | `ghosty agents create --name <name> [--engine <engine>] [--model <model-id>] [--prompt-file PROMPT.md] [--env K=V,…] [--workspace <slug>] [--channels teams=off] --json` → `id`. A model outside the engine answers 400 with the valid list. `--workspace` = born in that workspace, owned by its owner, active in its Teams |
| "set its identity / persona / system prompt" | write it with `references/identity.md` to a file, `ghosty agents set <id> --prompt-file PROMPT.md`, then `ghosty agents restart <id>` only if `needsRestart` |
| "let it see my Drive / use my connector" | `ghosty agents set <id> --connect google-drive` (owner only; the account must have it connected first). Files are the ones the owner picked in Conectores → Google Drive |
| "save / version this agent as a file", "change several settings at once" | `ghosty agents export <id> --out ./<name>` (agent.yaml + skills), edit the YAML, `ghosty apply ./<name> --dry-run`, show the plan (`+ ~ - !`), then `ghosty apply ./<name> --yes`. `--prune` only if the user wants removals; secrets stay as `${NAME}` unless set in the env |
| "make an agent like <other> / same setup as" | `ghosty agents create --name <n> --like <other-id> --dry-run` (lists what it copies: skills, toolsets, connectors, voice, databases, documents, board, shares — never prompt, env or channels), then without `--dry-run`, then give it its own prompt. `--board new` gives it its own board (source's columns), `--no-notify` shares without emailing people — ask the user which they want |
| "which toolsets / connectors does it really have?" | `ghosty agents get <id> --fields toolsets,connectors --json` → effective `toolsets[]` and `connectors[]` with `connected`, `granted`, `effective` |
| "turn off most of Ghosty's skills" | `ghosty skills house <id> --only a,b --dry-run`, then without `--dry-run` (`--enable`/`--disable` for a few) |
| "one board per number / move these cards to another board" | `ghosty boards ls <workspace>`, `boards create <workspace> "<name>" --columns-from main`, `boards assign <agent-id> "<name>"`, `boards move-cards <workspace> --from main --to "<name>" --integration <formmy-id> --dry-run` then `--yes` after confirming; `boards rm <workspace> "<name>" --yes` deletes an empty one |
| "edit its (long) prompt" | `ghosty agents get <id> --prompt-out PROMPT.md`, edit the file, `ghosty agents set <id> --prompt-file PROMPT.md` |
| "base its prompt on another agent's (a branch, a variant)" | write an overlay `{"edits":[{"what","find","replace","regex"?,"all"?}]}`, `ghosty agents set <id> --prompt-base <base-id> --prompt-overlay overlay.json --dry-run --out /tmp/p.md`, check, then without `--dry-run`. It regenerates when the base changes; its own prompt can't be edited while linked (`--prompt-base none` unlinks) |
| "give it the CRM / sales tools" | `ghosty agents set <id> --toolsets +crm` (a plain list replaces; `-x` removes) |
| "share it with / give access to <email>" | `ghosty agents share <id> <email> [--role editor\|admin]` (keeps ownership); `ghosty agents shares <id>` lists; `unshare` needs `--yes` |
| "turn off / on this skill" | `ghosty skills disable\|enable <id> <slug>` (its own or one of Ghosty's; `skills ls` shows `on`) |
| "delete this agent" | confirm with the user first, then `ghosty agents rm <id>` (owner only; `409` names the workspace where it is active) |
| "switch it to another engine" | `ghosty agents set <id> --engine <engine> [--model <model-id>] --dry-run` first (says what is forgotten and which machine is replaced; `ok:false` if it is busy), then the same without `--dry-run`. The model belongs to the NEW engine; each conversation keeps its history (gs passes it on its first turn) |
| "change the model" | `ghosty agents get <id> --json` (lists `models`), then `ghosty agents set <id> --model <model-id>` (restarts by itself) |
| "make it think more / less" | Ghosty · Lite: `ghosty agents set <id> --env GHOSTY_THINKING_EFFORT=off\|low\|medium\|high\|max`. Codex: `--env FLEET_EFFORT=none\|minimal\|low\|medium\|high\|xhigh\|max` |
| "give it these files / documents / knowledge" | `ghosty files put <id> <name> --file <local>`, one per file |
| "install / teach it a skill" | `ghosty skills add <id> <slug> --file SKILL.md` (or just `<slug>` from the catalog: `ghosty skills ls <id>`), then `ghosty agents restart <id>` only if `needsRestart` |
| "connect it to this MCP server" | `ghosty mcp get <id> --json > servers.json`, add the server, `ghosty mcp set <id> --file servers.json` (replaces; restarts by itself). One key per end user of another product: header `"Authorization": "Bearer ${turn.token}"` (http only, not ACP); each `message-stream` turn then needs `turnToken` |
| "what does it have?" | `ghosty agents get <id> --json` → prompt, model, files, skills, mcp |
| "does it work? / test it" | `ghosty try <id> "…" --json` → the agent's answer (see Verify) |
| "talk to it / ask it something" | `ghosty chat <id> "…" --json` → streams `chunk` lines, ends with `done` |
| "test it as a WhatsApp customer / with a photo / with earlier context" | `ghosty try <id> "…" --waba --session <made-up phone> [--media FILE] [--history FILE --reset] --json` → `sent[]` (what it would send, tools included: `kind` text/voice/file/rich with `rich` ubicacion/cita/boton/contacto/reaccion; nothing leaves). A customer attachment exactly as Formmy delivers it: add `--as-formmy` → `media.copied` |
| "show me this customer's chat on the board / did the attachment arrive?" | `ghosty cards ls <id> [--q name-or-phone] --json`, then `ghosty cards show <id> <folio\|phone> [--limit 50] --json` → `messages[]` with `role`, `content`, `mediaType`, `mediaCopied` |
| "replay these real conversations" | `ghosty try <id> --replay sample.json --out ./replay [--max N]`; read `./replay/replay.json`; `--resume` if it was cut |
| "is it configured right? / why does it answer badly?" | `ghosty agents doctor <id> --json` → `checks[]` with `level` (ok/warn/error) and a `fix` command each; exit 1 = something to fix |
| "did it forget the conversation? / did it start from scratch?" | `ghosty turns ls <id> --session <conversation> --json` → each turn's `sdk.resumed` (false + `freshReason` orphan/stale/replaced = it lost its memory; `new` = first turn). Claude agents only |
| "try my public demo as a visitor / does it remember after a box change?" | `ghosty demos try <slug> "…"` (same thread across runs; `--new` for a new visitor); memory across boxes: second message with `--recycle-between --agent <id>`. It counts against the demo's caps |
| "stop it / it keeps working in the background" | `ghosty turns cancel <id> <conversation-id> --json` → `detenido` + `background` (video jobs, subagents, wake-ups canceled: nothing more is delivered) |
| "test it with this photo / PDF" | `ghosty try <id> "…" --media FILE` (repeatable) or `ghosty chat <id> "…" --media FILE`; as a WhatsApp customer add `--waba --session <phone>` |
| "do X to all agents that…" (staff) | `ghosty agents ls --sponsor <email> \| --owner <email> [--engine E] --ids`, then `xargs -I{} ghosty agents set {} … --dry-run` first |
| "clean up its history / delete the test chats" | `ghosty conversations rm <id> --kind test --before 7d --dry-run`, confirm, then without `--dry-run` (`--yes`); customer threads: `conversations archive <id> --kind channel` (never rm) |
| "merge my duplicated documents" | `ghosty agents docs dedupe --dry-run`, show the merges, then `--yes`; two specific ones: `ghosty agents docs dedupe <keep-id> <drop-id>` |
| "use this file from my files in the chat" | `ghosty me files ls --q <name> --json` → id, then `ghosty chat <id> "…" --file <file-id>` |
| "which workspaces do I have / where is X's workspace" | `ghosty spaces ls [--user <email>] --json` (others: staff) |
| "create a workspace" | `ghosty spaces create <slug> --dry-run`, then without `--dry-run` (yours, trial; same rules as the web: confirmed email, plan cap). `--combo` other than teams: staff |
| "bring only the ad leads from this number" | `ghosty channels whatsapp history <id> --integration <formmy-id> --dry-run --list --only-ads` → show which, then without `--dry-run` (`--yes`); new numbers: `channels whatsapp link … --ads-only on` |
| "why did it fail? / it didn't answer" | `ghosty turns ls <id> --errors --since 24h --json` → failed turns with their `error`; a WhatsApp customer got NO turn at all: `ghosty turns ls <id> --skipped --since 24h --json` → `reason` (manual_mode, records_only, paused, reaction, channel_off, no_quota) |
| "what did it answer to that? / audit a reply" | `ghosty turns show <id> [turn-id] --json` → `input` (what came in) and `output` (what it answered), any engine or channel |
| "why was it slow? / where did the time go?" | `ghosty turns show <id> [turn-id] --json` → `byTool` (seconds per tool), `steps` (timeline), `totals.outputTokens`; pool engines only (`timeline: false` otherwise) |
| "is it running the new image? / it still behaves old" | `ghosty agents box <id> --check <path-the-new-image-brings> --json` → `stale`; if > 0, `ghosty agents box <id> --recycle`, one turn, check again (ACP: `agents restart`) |
| "which PDF templates does it have?" | `ghosty skills templates <id> --json` → `templates[]` with `source` (`agente:<skill>` or `casa`); its own go in `<skill>/pdf-templates/<name>.html` |
| "give it this database / what can it write" | `ghosty dbs ls <id>`, `ghosty dbs tables <id> <db>`, `ghosty dbs grant <id> <db> [--external-write t1,t2] [--external-rows t:col+col]` (tables it may write from WhatsApp/Messenger; tables where each customer sees only their own rows, matched by phone column — use it for any customers/orders table) |
| "are the catalog photos permanent? / photos stopped showing on WhatsApp" | `ghosty dbs tables <id> <db>` → per photo column permanent · external · empty; external ones: `ghosty dbs rehost <id> <db> --table T --column C` |
| "here are the photos the customer sent" (a folder named by SKU) | `ghosty dbs photos put <id> <db> --table catalogo --key-column sku --dir <folder> --dry-run`, show which SKUs have no row, then without `--dry-run` |
| "remove these env variables" | `ghosty agents set <id> --unset-env K1,K2` |
| "which models can it use? / is this model valid?" | `ghosty engines ls --models --json` → `engines[].models[]` (`id`, `multiplier`, `ownKeyOnly`); an invalid `--model` is rejected with the valid list |
| "how much did this workspace use / which agent spends most / messages left without balance" | `ghosty usage --workspace <slug> --by agent,day --days 14 --json` → `rows`, `agents` (turns, failed, billable, noQuota); the unanswered ones: `ghosty turns ls --workspace <slug> --quota-hits --json` (owner or staff) |
| "give this agent its own token bag / make it one-time" | staff only: `ghosty bags ls <workspace>`, `ghosty bags create <workspace> "<bag>" --tokens 20M [--recurring off]`, `ghosty agents set <id> --bag "<bag>"`; change: `ghosty bags set <workspace> "<bag>" --tokens 30M` |
| "renew / extend this workspace" or "rename its slug" | staff only: `ghosty spaces renew <workspace> --until YYYY-MM-DD --dry-run`, confirm, then `--yes`; `ghosty spaces rename <old> <new> --dry-run` first |
| "check X's health / X's app is stuck" (support) | staff only: `ghosty agents doctor --user <email> [agent]` — doctor, latest turns with «fetched» (`never` = their app didn't pick up a finished answer), plan/bag and boxes (duplicates flagged); `ghosty agents ls --user <email> --app android` shows their list and phones with the app build (old build = ask them to update) |
| "what did this agent run / did it leak anything" (support) | staff only: `ghosty conversations show <agent> <conversation> --tools` — every tool with its command (secrets masked) and the external domains it named; never while a turn runs there. Who sees a thread: `--why`; someone's threads: `ghosty conversations ls --user <email>` |
| "are images being metered / how much do images cost on Free" | staff only: `ghosty usage --images --plan free --since 7d [--by user]` — charges vs images agents delivered (far more deliveries than charges = the meter isn't reporting) and stored cost vs the table |
| "did the push reach them / did the announcement go out" | staff only: `ghosty push log --user <email> --since 2d` (announcements show as `novedad`); `ghosty novedades ls` shows delivered/rejected per announcement |
| "turn this tool off for customers / in WhatsApp" | `ghosty agents get <id> --fields tools --json` → `tools.canales`, then `ghosty agents set <id> --tool wa:<tool>=off` (no channel = everywhere) |
| "put this face / avatar on the agent" | `ghosty agents set <id> --avatar <https-url or local png/jpg/webp>` → updates every Teams where it's active |
| "remove this agent from the workspace" | irreversible (memory and files go too): confirm with the user, then `ghosty agents rm <id> --from-workspace --yes` (turns it off in every Teams, then deletes it) |
| "how many boxes / how much capacity does it have?" | `ghosty agents box <id> --json` → `capacity`; per workspace: `ghosty usage --boxes --workspace <slug> --json` |
| "apply the new plan now" | `ghosty plan apply <workspace> --dry-run`, show the changes and monthly total, confirm, then without `--dry-run` (`--yes`) |
| "copy its data from EasyBits" | `ghosty dbs import <id> <db> --from easybits:<db-id> --dry-run` first (needs `EASYBITS_API_KEY`), show the plan (and `gsOnly`: rows written in gs that replacing would delete — then use `--tables` or `--append`), then run without `--dry-run` (`--yes` if replacing rows) |
| "rename this database / delete an unused one" | `ghosty dbs rename <id> <db> <new> --dry-run --json` → `references` (prompts, env, skills it rewrites too), confirm, then without `--dry-run`; unused: `ghosty dbs ls <id> --orphans`, `ghosty dbs rm <id> <db> --dry-run`, confirm with the user, then `--yes` |
| "let it read / send this document" | `ghosty agents docs ls <id>`, `ghosty agents docs grant <id> "<name>" [--write] [--no-deliver]`; a new one: `ghosty agents docs add <id> --url <url> \| --file <path> [--description T]` (uploads and grants); several: `grant <id> "A" "B"` or `--like <other-id>` |
| "update / replace the catalog (or any document)", "go back to the previous one" | `ghosty agents docs replace <id> "<name>" --file <path> \| --url <url> [--note T]` — a NEW VERSION of the same document: keeps its name, access and every agent that uses it (never `add` a second copy: prompts name it and the other agents would keep the old one). `ghosty agents docs history <id> "<name>"` shows versions and who uses it; `ghosty agents docs restore <id> "<name>" <n>` |
| "answer with voice notes" | `ghosty agents set <id> --voice elevenlabs:<voice-id>\|kokoro:em_santa --voice-replies auto` |
| "answer in this WhatsApp group / make the group team or customer" | personal WhatsApp only: `ghosty channels wa groups ls <id> --json`, then confirm with the user and `ghosty channels wa groups enable <id> "<group>" --role equipo\|cliente --yes` (`--dry-run` first); a newly enabled group only answers when mentioned — `--wake all` to answer every message |
| "move / migrate this WhatsApp number from EasyBits or Formmy" | Business: find its Integration id and who answers it today with `ghosty channels whatsapp integrations <id> --json`, then `ghosty channels whatsapp link <id> --integration <formmy-id> --answer on\|off --dry-run --json` → show `plan`; confirm with the user (on = answers real customers), then without `--dry-run` and `--yes`; back: `unlink`. Its recent cards/messages/orders: `ghosty channels whatsapp history <id> --integration <formmy-id> --days 7 --dry-run` (works before linking), then without `--dry-run` (`--yes`; re-running never duplicates; it retries by itself during a deploy). It reports conversations that arrive paused by Formmy: list them with `ghosty cards ls <id> --paused --limit 200` and tell the user the agent won't answer those until they're resumed. Personal: `ghosty channels wa import <id> --from easybits:<agent-id> --own-number yes\|no --dry-run` (needs `EASYBITS_API_KEY`; prints whether the origin session is alive — if not, pair again instead), or `--from-file F` (EasyBits JSON) or `--from-dir DIR` (Baileys multi-file folder) `--own-number yes\|no --dry-run` (the user turns off EasyBits WITHOUT logout first), then `wa status` |
| "answer this WhatsApp number" | confirm with the user (real customers), then `ghosty channels whatsapp enable <id> <number> --yes`; `--dry-run` shows the plan |
| "save my ElevenLabs / MercadoPago key" | never put the key in a command: ask the user to run `ghosty credentials set <provider>` themselves (hidden prompt), or use `--from-env VAR` |
| "remove this skill" | `ghosty skills rm <id> <slug> --yes` keeps a local copy and returns `restore`; `ghosty skills get <id> <slug> --out DIR` to back one up first (without `--out` it only lists) |
| "how much have I used? / am I out of usage?" | `ghosty usage --json` → `plan.name`, `week.pct` / `month.pct` (0–1), `exhausted` |
| "let people try it without an account / share a demo" | `ghosty agents demo <id> --slug <name> [--expires "YYYY-MM-DDTHH:MM"] [--welcome T] [--starter T] [--chips "a\|b"] --json` then `ghosty agents demo enable <id>` → `url`; no flags = status; `--rotate` if the link leaked, `--off` to stop |
| "what files are in my account?" | `ghosty me files ls [--kind document] --json`; `me files upload <path>`, `me files rm <file-id>` |
| "what do my agents remember about me?" / "make it remember X" | `ghosty me memories ls --json`; `me memories add "X"` (app-wide, all agents), `edit <id> "…"`, `rm <id>` (only travels in personal chats) |
| "clean up the board / archive test cards" | `ghosty cards archive <id> --integration X \| --column X \| --before 30d --dry-run` first, then with `--yes` after the user confirms (reversible) |
| "which conversations does it have?" | `ghosty conversations ls <id> --json`; read one with `ghosty conversations show <id> <conv-id> --json` (`messages[]` with `role`, `text`); continue one with `ghosty chat <id> "…" --conversation <conv-id>` |
| "what's in its database? / fix this row" | `ghosty dbs query <id> <db> "SELECT …" [--arg V]… --json` (read only); `--write` only for a change the user asked for |
| "stop giving it this database" | `ghosty dbs revoke <id> <db>` (prints the `grant` that undoes it) |
| "give an agent to each of MY customers (partner)" | not for a normal owner: see https://www.ghosty.studio/docs/cli/partners.md (`ghosty partner --help`). Issue a credential without printing it: `ghosty partner credentials create --label prod --env-file .env.partner`; debug tools: `ghosty partner try <org> "…" --verbose`; several agents per partner: `--agent <slug>` on `partner agent get|apply` and `partner try`; a business's WhatsApp number: `channels whatsapp link <partner-agent-id> … --partner-tenant <org> --partner-agent <slug>`; WhatsApp groups of a partner agent: `ghosty partner wa groups ls|link <jid> --tenant <org> --agent <slug>|create|unlink`; a business's WABA number registered by the partner in Formmy: `ghosty partner waba links create --tenant <org> --agent nik-public` (url + secret for Formmy) then `set <id> --integration <formmy-id> --on` |
| "work in my client's account / workspace" | `ghosty login --device --profile <client>`: give the client the printed link and code to approve; then add `--profile <client>` to every command (`ghosty logout --profile <client>` revokes it) |
| "have it speak in this Teams room / thread at 9" | `ghosty schedule add <id> "…" --teams <workspace>#<room>[/<thread-message-id>] --at ISO \| --in 30m` → it answers there as if mentioned; `ghosty schedule ls\|rm <id> [sched-id] --teams <workspace>` |
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
- **Destructive commands need `--yes`** when you run them (no terminal): `agents rm`, `files rm`, `skills rm`, `credentials rm`, `board archive`, `dbs import` over existing rows, `channels whatsapp enable`, `channels wa groups enable`, `channels whatsapp link|unlink|history`, `channels wa import`, `agents unshare`, `boards move-cards`. Confirm with the user BEFORE adding it; prefer `--dry-run` first where it exists.

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
