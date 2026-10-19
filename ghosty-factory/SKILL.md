---
name: ghosty-factory
description: Prepare and maintain a GitHub repo for the Ghosty Software Factory (@plan, @build, @check agents working from Ghosty Teams) — AGENTS.md, the docs/agents/ knowledge base, .ghosty/factory.md (which agent and model plays each role) and small, reviewable PRs. Use when the user mentions Ghosty Factory, Software Factory, @plan/@build/@check, .ghosty/factory.md, docs/agents or wants their repo "ready for agents". Also use it to DELEGATE work from your terminal to the factory with `ghosty factory run` ("hand this to the factory", "let @build do it", "open a factory run").
license: MIT
compatibility: Any git repository hosted on GitHub. Network access to https://www.ghosty.studio for the docs.
metadata:
  author: ghosty-studio
  version: "1.1"
---

# Get a repo ready for the Ghosty Software Factory

The Software Factory is three agents working a GitHub repo from a Ghosty Teams room: `@plan`
writes a plan a person signs, `@build` builds it in a draft PR, `@check` reviews it with another
model. A person signs the plan and the PR. The goal is **PRs a person reviews in two minutes**,
so everything here serves that. Docs: https://www.ghosty.studio/docs/fabrica.md (English:
`/en/docs/factory.md`).

Everything below lives **in the repo**. Work on a branch and open a PR; never push to the default
branch.

## 1. `AGENTS.md` (the index)

Create or complete it at the repo root. Only commands that really exist in `package.json` (or
the project's equivalent):

```markdown
# AGENTS.md

## Comandos
- `npm ci` — instalar
- `npm run typecheck` — tipos
- `npm test` — pruebas

## Arquitectura
3–6 lines: main folders and what lives in each.

## Convenciones
- Small changes with tests; one PR per request.

## Qué no tocar
- `.github/` (CI and repo rules): a person reviews changes there.
- Secrets and `.env*` files.

## Conocimiento
Short notes in `docs/agents/`: decisions, pitfalls and glossary the code doesn't say by itself.

- [Topic](docs/agents/topic.md) — what it covers, in one line
```

If the repo already has `CLAUDE.md`, say so at the top of `AGENTS.md` and don't duplicate it.

## 2. The knowledge base: `docs/agents/`

One short note per topic. Each answers **what**, **why it was decided that way** and **how to
apply it**. Write only what the code doesn't say by itself: decisions, pitfalls that cost time,
vocabulary. Add one line per note to the "Conocimiento" section of `AGENTS.md`.

The factory reads it every turn: `@plan` cites the notes it used, `@build` writes or updates a
note **in the same PR** when a change settles a convention or finds a pitfall, and `@check`
flags a PR that contradicts a note. When you (a coding agent outside the factory) change a
convention in this repo, do the same: update the note in your PR.

## 3. `.ghosty/factory.md` (optional): who plays each role here

Overrides the workspace team for this repo. Frontmatter per role; the body is conventions that
travel with every factory turn and override the generic rules:

```markdown
---
plan: { agent: Planner }
build: { agent: Builder, model: opus }
check: { agent: Reviewer, model: flash }
---
Conventions of this repo that @plan, @build and @check must know:
- Tests run with `pnpm test` (Vitest).
- Never touch already-applied migrations in `prisma/migrations/`.
```

- `agent`: name or id of a Studio agent in the workspace. Ask the user; don't invent names.
- `model`: alias (`opus`, `sonnet`, `fable` for Claude; `pro`, `flash` for DeepSeek; `sol`,
  `terra`, `luna` for Codex) or a full id. It must match that agent's engine.
- Roles you leave out use the workspace default.

## 4. What makes factory PRs pass on the first review

- **Tests that run by themselves** in CI on every PR (typecheck, lint, tests). The factory's
  "Prepare repo" can add a CI workflow; if one exists, keep it green.
- **A way to run the app** (`start`, `preview` or `dev` script) so each PR gets a preview and
  the verdict card gets screenshots.
- **Small PRs.** The platform marks a PR high-risk when it passes ~400 lines or touches auth,
  migrations, dependencies, public API or `.github/`. Split big work.
- **`CODEOWNERS`** covering `.github/`, so a person reviews CI changes.

## 5. Delegate from your terminal

You (Claude Code, Codex…) can hand work to the factory and keep going with the user. It runs in
the repo's Teams room with the same plan, approvals and card as if the user typed it there.

**Delegate** long, self-contained work with a clear "done": a feature behind a flag, a refactor
with tests, a migration with a checklist, a bug with a reproduction. **Keep it** when it needs
back-and-forth with the user, it's a few lines, or you're still figuring out what to build.

1. Write the request like a ticket: the goal, why, how we'll know it's done (a check a person can
   run), and the files or docs to read first. One request = one PR.
2. `ghosty factory run "<request>" --repo owner/repo --wait --workspace <slug> --json`. It returns
   at `plan_review` with `next` (the exact commands). If the repo is in several rooms, add `--room`.
3. Show the user the plan (`ghosty factory runs show <n> --full`) and **ask before signing**:
   `ghosty factory runs approve <n> --plan-version <N> --yes`. To send it back:
   `ghosty factory runs changes <n> --note "…" --yes`.
4. `ghosty factory runs watch <n> --json` until `pr_review` (or `escalated`: show the points and
   let the user choose `approve` = another loop, or `changes` = replan).
5. Show the PR and **ask before merging**: `ghosty factory runs merge <n> --yes`. If CI is still
   running it queues and goes in when it passes.

Exit code 1 from `--wait`/`watch` only means the timeout ran out; run `watch` again. The user can
always follow along in the room; `ghosty factory runs card <n>` brings the live card down.

## Verify

- Every command in `AGENTS.md` runs.
- Every note in `docs/agents/` is linked from `AGENTS.md` and every link resolves.
- `.ghosty/factory.md` frontmatter only uses `plan`, `build` and `check`, with `agent`/`model`.
- Tell the user they can check the result in Ghosty Teams → `/factory` → the repo →
  "Ready for agents" and "Team in this repo" (↻ refreshes it).
