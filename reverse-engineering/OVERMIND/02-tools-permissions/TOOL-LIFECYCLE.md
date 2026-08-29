# Tool Execution Lifecycle

**Status:** RECOMMENDED

## Current gap

`ToolRegistry.execute(call)` es deliberadamente pequeño: lookup, arguments validation, Tool.execute y normalized observation. Para un agente local es suficiente. Para permisos, streaming progress, cancellation, persistence y child tools conviene añadir un **ToolExecutor** alrededor, sin engordar ToolRegistry.

## ToolExecutionContext

Propuesta:

```text
ToolExecutionContext
- session_id
- execution_id
- call_id
- model_turn
- cancellation
- permission port
- event emitter
- metadata/progress reporter
- capability grants
```

No entregar Agent ni Runtime completos.

## Lifecycle

```text
model emits complete ToolCall
       |
       v
validate protocol/arguments
       |
       v
permission check
       |
       +-> denied -> ToolObservation(error)
       |
       v
FACT/SIGNAL tool.started
       |
       v
Tool.execute(context)
       |
       +-> progress SIGNALs
       |
       +-> side effect -> MutationJournal
       |
       v
FACT tool.completed / tool.failed
       |
       v
atomic ProtocolUnit committed by Agent
```

## Preserve atomicity

El lifecycle externo no debe provocar que una tool truncada/parcial se ejecute. `Agent` sigue aceptando solo complete normalized ToolCalls del Model layer.

## Hooks

Antes de copiar `before/after` hooks de OpenCode, usar Event observers. Solo introducir un hook mutante si existe un caso concreto donde observation no basta.

Ejemplo legítimo futuro: policy middleware que normaliza/redacta una outbound network request, con contrato explícito. No un hook global `on_anything`.

## Metadata

Overmind ya limita event metadata a 4096 bytes. Mantener bounded metadata y añadir referencias externas para outputs grandes si hace falta; no inflar FACTs con payload arbitrario.
