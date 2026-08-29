# System Prompt Assembly

**Status:** VERIFIED-CODE

No existe un único system prompt monolítico. El contexto de sistema se construye en varias capas.

## Provider/model prompt family

`SystemPrompt.provider(model)` selecciona un prompt base según `model.api.id` / provider:

- IDs con `muse` → `meta.txt` parametrizado;
- `gpt-4`, `o1`, `o3` → `beast.txt`;
- GPT con `codex` → `codex.txt`;
- otros GPT → `gpt.txt`;
- `gemini-` → `gemini.txt`;
- `claude` → `anthropic.txt`;
- `trinity` → `trinity.txt`;
- Kimi/Moonshot → `kimi.txt`;
- fallback → `default.txt`.

Esta selección es independiente del agent prompt.

## Environment block

`SystemPrompt.environment(model)` construye texto con:

- model name e ID exacto `provider/model`;
- working directory;
- workspace root;
- si el proyecto usa git;
- platform;
- fecha actual;
- project references con name/path/description cuando existen.

## Skills block

Si el permiso `skill` no está globalmente disabled para el agent, se obtiene `skill.available(agent)` y se añade un bloque que explica al modelo que debe usar la skill tool cuando una tarea coincide. El listado se formatea en modo verbose.

## MCP instructions block

Se fusionan agent permissions y session permissions. Solo se incluyen instrucciones de un servidor MCP si al menos una de sus tools asociadas sigue habilitada. Se serializan dentro de `<mcp_instructions><server name=...>`.

## Actual ordering in loop

En `SessionPrompt` el system array se monta como:

```text
environment
+ project/global instructions
+ MCP instructions (optional)
+ skills (optional)
```

Después `LLMRequestPrep`/provider transforms pueden añadir o reubicar información para un provider concreto. Por tanto, el array anterior es el system context lógico pre-provider, no necesariamente una lista de mensajes wire idéntica para todos los proveedores.

## Structured output addition

Cuando el user pide `json_schema`, el loop añade un system directive específico que obliga a usar `StructuredOutput`.

## Sources

- `packages/opencode/src/session/system.ts` — `d0c608b203f68f8c84f117129852b30c9b73d090`
- `packages/opencode/src/session/prompt.ts` — `0f85d44f...`
- `packages/opencode/src/session/llm.ts` — `a99f8acf...`
