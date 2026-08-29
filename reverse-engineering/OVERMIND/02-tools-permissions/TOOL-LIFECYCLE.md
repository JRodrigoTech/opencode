# Tool Lifecycle — external agent como Tool de alto nivel

## Diseño actual suficiente para el MVP

Overmind ya ejecuta complete normalized ToolCalls a través de ToolPort/ToolRegistry y produce `ToolExecutionResult` que Agent incorpora como ProtocolUnit.

Un agent externo puede encajar en ese contrato sin rediseñar Agent:

```text
model emits complete `agent.code` ToolCall
   |
   v
ToolRegistry validation
   |
   v
OpenCodeDelegateTool.execute
   |
   +-> spawn/ACP adapter
   +-> consume external activity internally
   +-> enforce timeout/cancel/policy
   |
   v
bounded ToolExecutionResult
   |
   v
atomic ProtocolUnit committed
```

## No remote-tool mirroring

La Tool no debe exponer el catálogo interno de OpenCode. El external agent ejecuta su propio loop.

## ToolExecutionContext

Un futuro `ToolExecutionContext` con cancellation/progress/events puede ser útil cuando varias Tools lo requieran. No es requisito previo para el spike si el adapter puede cumplir safely usando el contrato actual.

## Side effects inciertos

Si se cancela o pierde conexión mientras OpenCode ejecuta una mutación, no retry automático de la misión completa. Marcar resultado indeterminado/failed y permitir reconcile/resume explícito.

## Result size

Resumen bounded en `content`; details grandes en metadata limitada o artifact/reference. No volcar stdout JSON completo en ProtocolUnit.
