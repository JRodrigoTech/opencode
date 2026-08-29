# Architecture Overview

**Status:** VERIFIED-CODE + DERIVED  
**Baseline:** `production@df35e842f59bc115bb7c0479a8e11f017d443f2c`

## Executive model

OpenCode production se comporta como un runtime de agente stateful. El modelo LLM no posee directamente el workspace ni la persistencia. OpenCode le presenta contexto y tools, ejecuta esas tools bajo permisos, reduce el stream de resultados a estado persistente y decide si debe continuar otra iteración.

```text
user / client
    │
    ▼
SessionPrompt.prompt
    │  create User message + parts
    ▼
SessionPrompt.runLoop
    │
    ├─ select Agent / Model
    ├─ load compacted history
    ├─ apply reminders/instructions
    ├─ resolve tools
    ├─ build system context
    ▼
LLM.stream
    │
    ├─ AI SDK runtime
    └─ native @opencode-ai/llm runtime
    │
    ▼ normalized LLMEvent
SessionProcessor
    │
    ├─ reasoning/text parts
    ├─ tool call state
    ├─ usage/cost
    ├─ step snapshots/patches
    └─ retry/error/overflow
    │
    ├─ stop
    ├─ compact ──► SessionCompaction
    └─ continue ─► next loop iteration
```

## Core responsibilities

### `SessionPrompt`: orchestration

`packages/opencode/src/session/prompt.ts` es el principal coordinator. Expone `prompt`, `loop`, `shell`, `command`, `cancel` y resolución de prompt parts. El loop es deliberadamente iterative: una respuesta con tool calls no termina necesariamente el turno; los resultados de las tools vuelven al modelo en otra iteración.

### `SessionProcessor`: event reducer

`processor.ts` recibe un stream de `LLMEvent` y lo proyecta a `SessionV1.Part`: reasoning, text, tool, step-start, step-finish y patch. También actualiza el `Assistant` message, tokens y coste.

### `ToolRegistry` + `SessionTools`: capability plane

El registry inventaría tools built-in y custom. `SessionTools.resolve` convierte esas definiciones a `ai.Tool`, transforma schemas para el modelo/provider, añade contexto de ejecución, hooks de plugin, permisos y MCP.

### `Permission`: policy plane

No se evalúan permisos solo al registrar tools. Cada tool puede realizar `ctx.ask`; la evaluación utiliza reglas wildcard y la última coincidencia. Las aprobaciones `always` viven en el estado de la instancia y pueden resolver otras peticiones pendientes.

### `LLM`: model/transport seam

`session/llm.ts` prepara la llamada, selecciona runtime nativo cuando está habilitado y soportado, o AI SDK como camino por defecto. Ambos caminos terminan en el mismo vocabulario de eventos para el processor.

### Session persistence

La sesión almacena identidad, proyecto/workspace, parentID, agent/model, tokens/cost, permisos, revert y timestamps. El historial se modela como messages + parts; por tanto, tool execution y patches son parte de la historia, no logs externos.

## Architectural properties

1. **Event-normalized LLM boundary.** El processor no necesita acoplarse al wire protocol de cada proveedor.
2. **Capability resolution per turn.** Las tools visibles dependen de agent, model, provider, flags y session permissions.
3. **Context is assembled, not static.** System context combina environment, project instructions, MCP instructions, skills y transforms.
4. **Workspace mutation is observed.** Snapshots antes/después de steps permiten producir patches y revert.
5. **Sessions are concurrency domains.** `SessionRunState` mantiene un runner por session ID.
6. **Extensibility is interceptive.** Plugins pueden transformar definitions, messages, text, environment y tool lifecycle.
7. **Context length is managed semantically.** Compaction preserva una cola reciente y resume historia, además de podar tool outputs antiguos.

## Sources

- `packages/opencode/src/session/prompt.ts` — `0f85d44...`
- `packages/opencode/src/session/processor.ts` — `20aa8a84...`
- `packages/opencode/src/session/llm.ts` — `a99f8acf...`
- `packages/opencode/src/session/tools.ts` — `0f401c75...`
- `packages/opencode/src/session/session.ts` — `a2a91cd4...`
- `packages/opencode/src/permission/index.ts` — `2e27ff24...`
