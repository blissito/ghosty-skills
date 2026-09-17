# Recipe schema (v1.0.0)

Source of truth: `parseRecipe` in Ghosty Studio. Unknown keys are ignored, never rejected.

| Key | Type | Required | Notes |
|---|---|---|---|
| `version` | string | no | defaults to `"1.0.0"` |
| `title` | string | **yes** | trimmed, cut at 60 chars; becomes the agent's name |
| `description` | string | no | trimmed, cut at 200 chars; one line, quote it if it contains `: ` |
| `instructions` | string | no | the system prompt; use a block scalar (`|`) |
| `settings.goose_provider` | string | no | Goose-compatible; usually omitted |
| `settings.goose_model` | string | no | model id; must exist for the chosen engine |
| `extensions` | list | no | Goose extensions; kept as-is, Ghosty ignores them today |
| `x-ghosty.engine` | string | no | `ghosty-lite` · `goose` · `claude` · `codex` · `deepseek` |
| `x-ghosty.model` | string \| null | no | same as `settings.goose_model`; the panel reads either |
| `x-ghosty.channels` | object | no | `{ teams?: bool, web?: bool, whatsapp?: bool }`; default `{ teams: true }` |
| `x-ghosty.creatorVersion` | string | no | written by exports; do not set by hand |

## Validation checklist

1. Parses as YAML (no unquoted `: ` in scalars, consistent indentation, block scalar for `instructions`).
2. `title` present and non-empty.
3. `instructions` in second person, in the user's language, with the four headings.
4. No secrets.
5. If `settings.goose_model` is set, it is a real model id for that engine.

## Exporting an existing agent

`GET https://www.ghosty.studio/app/agents/<id>/recipe.yaml` (logged-in browser) downloads
`<name>.recipe.yaml` with the same shape, secrets stripped.
