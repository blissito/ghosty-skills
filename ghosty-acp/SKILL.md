---
name: ghosty-acp
description: Connect a code editor (Zed, VS Code, JetBrains, Neovim) to a Ghosty Studio agent over ACP (Agent Client Protocol) using the ghosty-acp bridge and the agent token. Use when the user wants to talk to their Ghosty agent from their editor, pastes an acp.agents or agent_servers config, or mentions ghosty-acp or ACP with ghosty.studio.
license: MIT
compatibility: Node 22+ on the user's machine (npx downloads the bridge); network access to the agent's wss:// address
metadata:
  author: ghosty-studio
  version: "1.0"
---

# Connect an editor to a Ghosty agent over ACP

A Ghosty Studio agent runs on its own machine behind a WebSocket. Editors launch a
**command** and speak ACP over stdio, so a bridge swaps the cable: `npx -y ghosty-acp <wss-url>`.
It has no dependencies and carries no secret; the token travels in an environment variable.

## What you need from the user

Both come from Ghosty Studio → **Agentes → their agent → "Desde tu editor (ACP)"** (a collapsed
section under the token card):

- the agent's ACP address: `wss://acp-<agentId>.<host>/acp`
- the token `gat_…` (the same token the `ghosty-agent` skill uses)

Never put the token in `args`: any process on the machine can read another process's
arguments. It goes in `env.GHOSTY_ACP_TOKEN`.

## Write the config for their editor

**Zed** — `Settings → settings.json`:

```json
{
  "agent_servers": {
    "Ghosty": {
      "type": "custom",
      "command": "npx",
      "args": ["-y", "ghosty-acp", "wss://acp-AGENT.../acp"],
      "env": { "GHOSTY_ACP_TOKEN": "gat_…" }
    }
  }
}
```

**VS Code** — install the *ACP Client* extension, then `settings.json`:

```json
{
  "acp.agents": {
    "Ghosty": {
      "command": "npx",
      "args": ["-y", "ghosty-acp", "wss://acp-AGENT.../acp"],
      "env": { "GHOSTY_ACP_TOKEN": "gat_…" }
    }
  }
}
```

Do not use VS Code's "Add Agent" wizard: it asks for command and args only, never the env
var, and the agent ends up without its key.

**JetBrains / Neovim** use the same triple (command, args, env) in their own ACP client
settings. `GHOSTY_ACP_CWD` optionally sets the remote working directory.

## When it does not connect

| Symptom | Meaning | Do |
|---|---|---|
| `rejected (1006)` right away | token mismatch (rotated?) | ask the user to copy the block again from the panel |
| slow first start (5–15 s) | the agent's machine was asleep; the first connection wakes it | wait, do not change anything |
| `preview host not found` | the machine was recycled; the platform recreates it on demand | retry once after ~5 s |
| `already serving N conversations` | every open session takes a slot | close a session in another client |

## Rules

- Node 22+ is required for `npx -y ghosty-acp`; check `node -v` if it fails to start.
- Read `ghosty-acp` in the user's editor config as the bridge, not as this skill.
- Never print the token back or commit the settings file with it inside.
