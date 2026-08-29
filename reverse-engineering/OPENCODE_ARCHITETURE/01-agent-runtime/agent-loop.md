# Agent Loop

**Status:** VERIFIED-CODE

## Entrypoints

`SessionPrompt.Service` expone:

- `prompt(input)`
- `loop(input)`
- `shell(input)`
- `command(input)`
- `cancel(sessionID)`
- `resolvePromptParts(template)`

El camino normal de chat entra por `prompt`, crea y persiste el user message y después llama a `loop`, salvo `noReply=true`.

## High-level state machine

```text
PROMPT RECEIVED
    │
    ├─ revert.cleanup(session)
    ├─ createUserMessage
    ├─ persist message + parts
    ├─ touch session
    └─ apply optional tool permission overrides
    │
    ▼
ensureRunning(sessionID)
    │
    ▼
while true
    │
    ├─ status = busy
    ├─ load compacted message history
    ├─ determine latest user/assistant/tasks
    ├─ early-finish check
    ├─ resolve agent + model
    ├─ handle queued subtasks
    ├─ apply reminders
    ├─ create assistant message
    ├─ create SessionProcessor handle
    ├─ resolve tools
    ├─ build model messages + system context
    ├─ processor.process(LLM stream)
    │
    ├─ result=stop    ─► break
    ├─ result=compact ─► create compaction; continue
    └─ result=continue──────────────► continue
```

## User message creation

`createUserMessage` decide agent/model/variant. Precedencia observada:

1. Agent explícito del input; si no, default agent.
2. Model explícito; si no, model del agent; si no, current model de la session.
3. Variant explícita; si no, variant del agent cuando exista en el modelo seleccionado.

Los parts se resuelven antes de persistir. Los file parts pueden convertirse en texto/attachments; los agent parts insertan una instrucción sintética para invocar `task`; MCP resources se leen y proyectan en text/file parts.

## Iteration completion

El loop no confía únicamente en `finish`. Si el último assistant tiene tool calls pendientes/no provider-executed, el ciclo debe continuar aunque el provider haya devuelto una razón equivalente a stop. Tool parts marcados como abortados/interrumpidos se excluyen de ese criterio para no revivir trabajo huérfano.

## Structured output

Si el último user message solicita `json_schema`, el loop añade `StructuredOutput` como tool y fuerza `toolChoice=required`. El tool captura el JSON validado y lo asigna como `assistant.structured`; si el modelo finaliza sin invocarlo, se produce `StructuredOutputError`.

## Subtasks

Los `SubtaskPart` se ejecutan mediante `TaskTool` desde el loop. OpenCode crea un assistant message/tool part que representa esa ejecución, aplica hooks before/after, usa el ruleset del subagent combinado con permisos de session y persiste el resultado antes de continuar.

## Side work

En la primera fase del loop se pueden lanzar summaries en background (`forkIn(scope)`). Al salir se invoca pruning de compaction también en background. Estos side effects no cambian la semántica principal de stop/continue/compact.

## Sources

- `packages/opencode/src/session/prompt.ts` — blob `0f85d44f...`
- `packages/opencode/src/session/tools.ts` — blob `0f401c75...`
- `packages/opencode/src/session/run-state.ts` — blob `5cefdd04...`
