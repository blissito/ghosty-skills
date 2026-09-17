---
name: ghosty-recipe
description: Write, review or convert a Ghosty Studio agent recipe (the portable .recipe.yaml with title, instructions, model and channels) so it can be uploaded in "Nuevo agente → Plantilla". Use when the user wants to create a Ghosty agent from a spec, turn a persona or job description into an agent, export/share an agent, or mentions a Ghosty recipe or recipe.yaml.
license: MIT
compatibility: No network needed to write a recipe; uploading it happens in the Ghosty Studio panel
metadata:
  author: ghosty-studio
  version: "1.0"
---

# Ghosty agent recipes

A recipe is a YAML file that fully describes an agent **without secrets**: who it is, how it
talks, which model, which channels. It is what Ghosty Studio exports from an agent
(`/app/agents/<id>/recipe.yaml`) and what "Nuevo agente → Plantilla → sube una receta"
imports. The format is Goose-compatible plus an `x-ghosty` block.

## Write one

```yaml
version: "1.0.0"
title: Nora, recepción de Dental Sur           # ≤ 60 chars, shown as the agent's name
description: Agenda citas y resuelve dudas de la clínica por WhatsApp.   # ≤ 200 chars, one line
instructions: |
  Eres Nora, asistente de la clínica Dental Sur.

  ## Cómo hablas
  Español de México, corto, sin diagnósticos.

  ## Qué sabes hacer
  - Agendar, mover y cancelar citas.
  - Explicar precios de la lista que tienes en tu workspace.

  ## Qué NO haces
  - No das diagnósticos ni recetas.

  ## Cuándo preguntas
  Sólo cuando la respuesta cambia el resultado.
settings:
  goose_model: claude-sonnet-5
x-ghosty:
  engine: ghosty-lite
  channels: { teams: true, whatsapp: true }
```

Field-by-field rules are in `references/schema.md`. Read it before validating a recipe.

## Rules

- **`title` is required**; everything else is optional but a recipe without `instructions` is
  useless. Write instructions in the user's language and in second person.
- **Never a colon followed by a space inside `description`** unless the value is quoted
  (`description: "Ventas: WhatsApp"`): unquoted it is a YAML parse error and the whole
  recipe is skipped.
- Structure `instructions` with the four headings the house templates use: *Cómo hablas*,
  *Qué sabes hacer*, *Qué NO haces*, *Cuándo preguntas* (plus *Formato* for Teams agents).
  Keep it under ~3,000 characters: it is prepended to every conversation.
- Engines: `ghosty-lite` or `goose` when the agent needs its own machine (files, skills,
  MCPs); `claude`, `codex`, `deepseek` for prompt-only agents. Models must be one the engine
  offers (`claude-sonnet-5`, `claude-opus-5`, `claude-haiku-4-5-20251001`…); when unsure,
  omit `settings` and let the panel pick.
- No tokens, keys, phone numbers or URLs with credentials in a recipe: it is meant to be
  pasted in an email.
- Prefer converting what the user already has (a job description, an old system prompt)
  over inventing; ask only for what changes the result.

## Deliver

Give the YAML in a fenced block and tell the user: save it as `<name>.recipe.yaml`, then in
Ghosty Studio → **Agentes → Nuevo agente → Plantilla → sube una receta**. To tune it after
creation, use the `ghosty-agent` skill (needs the agent token).
