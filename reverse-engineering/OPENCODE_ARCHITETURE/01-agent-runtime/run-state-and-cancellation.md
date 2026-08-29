# Run State and Cancellation

**Status:** VERIFIED-CODE

## One runner per session

`SessionRunState` mantiene un `Map<SessionID, Runner>`. `runner(sessionID, onInterrupt)` devuelve el existente o crea uno nuevo. Esto convierte a la session en dominio de serialización/cancelación.

Callbacks del Runner:

- `onBusy` → `SessionStatus = busy`
- `onIdle` → elimina el runner del map y pone `idle`
- `onInterrupt` → obtiene el último assistant/result de fallback

## `ensureRunning`

El loop normal usa `ensureRunning(sessionID, onInterrupt, work)`. La intención es reutilizar la ejecución si corresponde y centralizar el ownership de la fibra de trabajo.

## Shell exclusivity

`startShell` usa el mismo runner y traduce `RunnerBusy` a `Session.BusyError`. Operaciones de revert también llaman `assertNotBusy`, por lo que no compiten con un agent run activo.

## Cancellation semantics

`cancel(sessionID)` realiza dos acciones:

1. cancela background jobs relacionados;
2. cancela el runner activo si existe.

Si no existe runner, fuerza status `idle`.

### Background job traversal

La cancelación no se limita a jobs cuyo ID sea exactamente la session. Recorre jobs running y considera relaciones mediante metadata:

- `metadata.sessionId`
- `metadata.parentSessionId`

Los jobs cancelados se añaden al conjunto `pending`, lo que permite encontrar descendientes transitivos en iteraciones posteriores.

## Processor interruption

La interrupción del stream provoca que `SessionProcessor` marque tool calls no asentadas como error/interrupted y complete el assistant. En `SessionPrompt`, un abort también puede convertir el assistant a un error `AbortError` con timestamp final.

## Invariant

Una cancelación debe dejar persistencia en estado interpretable: no se permite que un `ToolPart` running quede indefinidamente como trabajo pendiente normal.

## Sources

- `packages/opencode/src/session/run-state.ts` — `5cefdd04a3f3bb712c67c1f89687644a633509d0`
- `packages/opencode/src/session/processor.ts` — `20aa8a84...`
- `packages/opencode/src/session/revert.ts` — `03e5afd0...`
