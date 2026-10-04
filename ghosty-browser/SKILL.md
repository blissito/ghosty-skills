---
name: ghosty-browser
description: Drive the user's own Chrome (with the sessions they already have open) from a coding agent (Claude Code, Cursor, Codex) or script through the Ghosty for Chrome extension — with the @ghostystudio/browser-mcp MCP server (local native bridge, no token) or the Ghosty Studio HTTP API (/api/browser/call with a personal bt_ token). Use when the user wants their Ghosty agent or Claude Code to act in their logged-in browser, mentions Ghosty for Chrome, browser_* tools, ghosty.studio/chrome, or a bt_ token.
license: MIT
compatibility: Needs curl or Node 18+, network access to https://www.ghosty.studio, and Chrome with the Ghosty extension signed in to ghosty.studio
metadata:
  author: ghosty-studio
  version: "1.1"
---

# Use the person's Chrome through Ghosty

**Ghosty for Chrome** lends the user's browser to an agent: it opens pages, reads, fills forms,
clicks and takes screenshots inside the sessions the user already has. The agent decides, the
extension executes. The user sees a lilac border, a named cursor and a **■ Stop** button; their
mouse never moves. Full guide: https://www.ghosty.studio/docs/navegador (EN:
https://www.ghosty.studio/en/docs/browser).

## Setup (the user does this once)

1. Install from https://www.ghosty.studio/chrome (the Chrome Web Store listing is on its way;
   that link always points to the current install page).
2. Sign in to https://www.ghosty.studio in that Chrome and open `/c`: the extension pairs with
   the account by itself. Its panel (⌘⇧G / Ctrl+Shift+G) says **Connected**.

There is one connection per person; the last Chrome to connect wins.

## From a Ghosty agent (chat, iOS/Android app, scheduled tasks)

Nothing to configure. While the owner's extension is connected, their agent gets `browser_*`
tools in **personal chats and scheduled tasks** (never in customer channels like WhatsApp,
Messenger or embed). Ask in plain language: "Open my Shopify dashboard and list today's pending
orders". Saved shortcuts: "save this as shortcut /prices: …", then `/prices`.

## From Claude Code, Cursor or Codex (MCP server)

`@ghostystudio/browser-mcp` exposes every tool as `browser_*`.

```bash
npx -y @ghostystudio/browser-mcp install-host   # once, on the computer where Chrome runs (no token)
claude mcp add ghosty-browser -- npx -y @ghostystudio/browser-mcp
codex mcp add ghosty-browser -- npx -y @ghostystudio/browser-mcp
# Cursor: ~/.cursor/mcp.json → {"mcpServers":{"ghosty-browser":{"command":"npx","args":["-y","@ghostystudio/browser-mcp"]}}}
```

After `install-host` the user reloads the extension (`chrome://extensions` → ↻). `browser_status` shows
`localBridge: true`. For Chrome on another machine, set `GS_BROWSER_TOKEN` (remote path, below).

## From a script or your own server (HTTP)

1. **Token**: the extension panel shows a personal `bt_…` token (from `GET /api/browser/token`
   with the web session). It lasts 30 days and only reaches that person's browser. Ask the user
   to paste it into an env var (`GS_BROWSER_TOKEN`); never print it back or commit it.
2. **Status and tools**:

   ```bash
   curl -s https://www.ghosty.studio/api/browser/call -H "Authorization: Bearer $GS_BROWSER_TOKEN"
   ```

   → `connected`, `email`, `lastStep`, `tools[]` (`name`, `description`, `inputSchema`). Expose
   each as an MCP tool if you are writing a server.
3. **Run a tool**:

   ```bash
   curl -s https://www.ghosty.studio/api/browser/call \
     -H "Authorization: Bearer $GS_BROWSER_TOKEN" -H "content-type: application/json" \
     -d '{"tool":"navigate","input":{"url":"https://example.com"},"client":"Claude Code"}'
   ```

   `tool` accepts `navigate` or `browser_navigate`. `client` is the name the user sees in the
   panel. `timeoutMs` defaults to 60000 (max 180000). `200` = result, `409` = browser not
   connected, `504` = the tool failed or timed out (`error`).

OpenAPI: https://www.ghosty.studio/openapi.yaml (tag *Navegador*).

## Driving pages reliably

| Step | Tool |
|---|---|
| Open a URL (waits for load) | `navigate`, `navigate_back` |
| Look before acting: structure with a `ref` per element | `read_page` (`boxes: true` adds coordinates) |
| Find one element on a big page by description | `find` ("the Publish button in the modal") |
| Read long content without menus | `get_page_text` |
| Act on a fresh `ref` | `click`, `type`, `fill_form`, `select_option`, `press_key`, `hover`, `drag` |
| Scroll or wait for text | `scroll`, `wait_for` |
| See the page | `take_screenshot` (to look, not to pick coordinates) |
| Canvas, maps, odd widgets | `computer` (mouse/keyboard by coordinates) |
| Attach without a file picker | `attach_image`, `file_upload` |
| Tabs and window | `tabs`, `resize`, `viewport` |
| Debug the user's site | `console_messages`, `network_requests` |
| Record what you did | `gif_creator` (`start_recording` → … → `stop_recording` → `export`) |

- **Parallel work: always pass `tabId`.** `tabs {action:"new", url, background:true}` returns a
  `tabId`; pass it to every call of that task. Two agents/subagents (say, YouTube and TikTok) then
  run at once without stepping on each other or stealing the visible tab. Answers start with `[tab N]`.
- **Uploads**: `read_page` ends with "Subir archivos" listing every `<input type=file>` (also the
  HIDDEN ones behind a button) with a `ref`; pass it as `target` to `file_upload` with absolute paths.
- **Accounts/channels**: switch with the page's own account picker (avatar → "Switch account"; on
  Google also `?authuser=N` or accounts.google.com/AccountChooser). Never type passwords.
- Every tool has a hard timeout: a stuck call returns an error instead of blocking the next ones.
- **Refs go stale.** After a navigation or a re-render, call `read_page` or `find` again before
  the next click. Never guess a ref.
- The agent only sees the tabs in the **"Ghosty"** tab group; leave the user's other tabs alone.
- **It never types passwords.** On a login page the user gets a notification; ask them to sign
  in and continue when they say so.
- Confirm before anything irreversible (publish, pay, send, delete) unless the user asked for
  exactly that.
- If the user hits Stop, stop and ask; do not retry on your own.
- On `409` / "not connected": tell the user to open Chrome with the Ghosty extension and their
  ghosty.studio session (opening `/c` pairs it), and give https://www.ghosty.studio/chrome if
  they don't have it.

Privacy: https://www.ghosty.studio/privacidad#seccion-11
