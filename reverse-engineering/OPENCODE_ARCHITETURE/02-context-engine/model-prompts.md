# Model-specific Prompts

**Status:** VERIFIED-CODE for routing; prompt-body semantics require per-file reading when changed.

## Routing table

| Condition | Prompt asset |
|---|---|
| `api.id` contains `muse` | `session/prompt/meta.txt` |
| contains `gpt-4`, `o1`, `o3` | `session/prompt/beast.txt` |
| contains `gpt` + `codex` | `session/prompt/codex.txt` |
| contains other `gpt` | `session/prompt/gpt.txt` |
| contains `gemini-` | `session/prompt/gemini.txt` |
| contains `claude` | `session/prompt/anthropic.txt` |
| contains `trinity` case-insensitive | `session/prompt/trinity.txt` |
| contains `kimi`, or provider `kimi-for-coding`/`moonshotai`/`moonshotai-cn` | `session/prompt/kimi.txt` |
| otherwise | `session/prompt/default.txt` |

## Why this boundary matters

OpenCode intenta mantener un agent contract uniforme sobre modelos con capacidades y tool conventions diferentes. La existencia de familias de prompt demuestra que parte de la compatibilidad se implementa en prompt engineering y no solo en schemas/provider transforms.

Debe distinguirse:

1. **Agent prompt** (`Agent.Info.prompt`) — comportamiento de un rol concreto como explore/compaction.
2. **Model prompt** — instrucciones base específicas de familia de modelo.
3. **Environment/instructions/skills/MCP** — contexto dinámico.
4. **Provider transform** — adaptación del formato wire/provider options.

## Audit invariant

No se debe afirmar que dos modelos reciben “el mismo prompt” solo porque comparten Agent. Incluso con el mismo historial y tools, pueden diferir el model-specific base prompt y las transformaciones de mensajes/provider options.

## Source

- `packages/opencode/src/session/system.ts` — `d0c608b...`
- assets en `packages/opencode/src/session/prompt/*.txt`
