# Ghosty Studio agent skills

Skills in the [Agent Skills](https://agentskills.io) format for coding agents (Claude Code, Cursor,
Codex, Copilot…). Install one or all:

```bash
npx skills add blissito/ghosty-skills
# or straight from the site (same skills, discovered via /.well-known/agent-skills):
npx skills add https://ghosty.studio
```

| Skill | What for |
|---|---|
| `ghosty-agent` | configure a Ghosty Studio agent through its API: identity, model, files, skills, MCP servers |
| `ghosty-docs` | read the Ghosty docs from an agent: markdown pages, `llms.txt`, the docs MCP server |
| `ghosty-acp` | connect Zed, VS Code, JetBrains or Neovim to the agent over ACP |
| `ghosty-recipe` | write or review an agent recipe (`.recipe.yaml`) to upload in the creator |

This directory is mirrored from `public/skills/` in
[blissito/ghosty-studio](https://github.com/blissito/ghosty-studio). Docs: https://www.ghosty.studio/docs/configurar
