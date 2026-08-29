# Tool Lifecycle

**Status:** VERIFIED-CODE

## Lifecycle across components

```text
LLM emits tool-input-start
        │
        ▼
Processor creates pending ToolPart
        │
        ├─ input deltas accumulate
        ▼
LLM/tool runtime emits tool-call
        │
        ▼
ToolPart -> running
        │
        ├─ before hook
        ├─ permission asks as needed
        ├─ implementation executes
        └─ metadata updates may stream
        │
        ├─ success -> tool-result -> completed
        └─ failure -> tool-error  -> error
```

## Persistent transition semantics

El processor no guarda solo resultado final. Las transiciones se escriben mediante `session.updatePart`, por lo que un cliente puede observar progreso y reconstituir estado.

## Metadata during execution

`Tool.Context.metadata(val)` actualiza el ToolPart running con title/metadata/input/time. Si la tool ya no está running/pending, la actualización se ignora.

## Attachments

Un completed tool result puede contener attachments. El processor normaliza especialmente imágenes y conserva metadata de truncation/normalization.

## Aborted tools

Durante cleanup, una tool que no terminó se transforma en:

- status `error`;
- error `Tool execution aborted`;
- end timestamp;
- `metadata.interrupted=true`.

Esta decisión evita dejar estados running fantasma después de cancelación/error.

## Provider-executed calls

Algunos workflows/proveedores pueden marcar tool calls como `providerExecuted`. El loop considera este metadata para decidir si necesita otro ciclo local.

## Sources

- `packages/opencode/src/session/processor.ts`
- `packages/opencode/src/session/tools.ts`
- `packages/opencode/src/tool/tool.ts`
