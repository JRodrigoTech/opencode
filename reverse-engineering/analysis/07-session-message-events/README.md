# 07 — Sessions, mensajes, eventos y persistencia

## Objetivo

Este dossier reconstruye la evolución y el comportamiento vigente de los subsistemas de **Session**, **Message**, **state/steps**, **event streams**, **persistence**, **replay**, **resume**, **recovery** y **sincronización cliente/backend** de OpenCode.

La baseline funcional usada es `dev`. Las branches históricas se emplean para reconstruir decisiones de diseño, transiciones y alternativas, pero no se asume que una branch siga vigente sólo por existir.

## Metodología

La investigación combina cuatro tipos de evidencia:

1. **Código actual de `dev`**: fuente principal para describir comportamiento vigente.
2. **Commit nominal de una branch**: preferido cuando la branch ya está absorbida o tiene un diff global ruidoso.
3. **Diff de branch contra `dev` o merge-base**: usado cuando el cambio es estrecho y atribuible.
4. **Naming de branch**: evidencia secundaria; nunca se usa por sí solo para afirmar un comportamiento interno.

Cuando una branch es long-lived y acumula cientos o miles de commits no relacionados, se evita atribuir el diff completo a la feature nominal. En esos casos se inspeccionan commits concretos y paths del dominio.

### Convenciones de evidencia

- **Hecho confirmado**: demostrado por código actual, schema, test o commit concreto.
- **Inferencia**: interpretación arquitectónica derivada de varias evidencias; se marca como tal.
- **Histórico**: comportamiento probado en una branch/commit, pero no necesariamente vigente en `dev`.

## Resumen ejecutivo

### Arquitectura reconstruida

El subsistema actual puede resumirse así:

```text
provider / LLM stream
        |
        v
Session processor / runner
        |
        +---- eventos live-only (deltas) --------------------+
        |                                                     |
        v                                                     v
SessionEvent durable                                   clientes live
        |
        v
EventV2.commit()
  [transacción SQLite]
        |
        +--> asigna seq por aggregate/session
        +--> valida orden, replay e idempotencia
        +--> ejecuta projectors
        +--> actualiza event_sequence
        +--> inserta event
        |
        v
proyecciones SQLite
  - session / message / part        (compatibilidad V1)
  - session_message / session_input (timeline/admission V2)
        |
        +--> SessionStore / Session APIs
        +--> history/events durable stream
        +--> bridge legacy + sync
        v
backend / SDK / TUI / desktop / otros clientes
```

### Invariantes principales

1. **El orden durable de una Session se define por `seq` del aggregate, no por timestamps ni por el orden lexicográfico del ID.**  
   Referencias: `packages/core/src/event.ts`, `packages/core/src/event/sql.ts`, `packages/core/src/session/sql.ts`.

2. **Los fragmentos de streaming no son necesariamente durables.** `Text.Delta`, `Reasoning.Delta`, `Tool.Input.Delta` y `Compaction.Delta` son live-only; los eventos `Ended` contienen el valor completo replayable.  
   Referencia: `packages/schema/src/session-event.ts`.

3. **El commit durable y sus proyecciones ocurren en la misma transacción SQLite.** Por tanto, el event log y las proyecciones registradas no deberían quedar desalineados por un commit parcial.  
   Referencia: `packages/core/src/event.ts`.

4. **`parentID` y fork no significan lo mismo.** `parentID` expresa una relación parent/child entre Sessions; `Session.fork()` clona historia en una Session nueva e independiente y remapea IDs internos.  
   Referencia: `packages/opencode/src/session/session.ts`.

5. **El estado de ejecución `busy/retry/idle` es transitorio.** Se mantiene en memoria y se publica como eventos de estado, pero no es el event log durable de la conversación.  
   Referencias: `packages/opencode/src/session/run-state.ts`, `packages/opencode/src/session/status.ts`.

6. **Message ID y Event ID son dominios distintos.** La evolución `refactor/core-v2-message-identity` introdujo `msg_*` separado de `evt_*` y conversiones deterministas entre ambas identidades.  
   Commit: `f2d133edf7430f3229a84c29d6b0e1690f1f2a60`.

7. **El replay durable es estricto e idempotente.** Un evento repetido con `seq <= latest` sólo se acepta si ID, tipo y datos coinciden exactamente; una diferencia se considera divergencia.  
   Referencia: `packages/core/src/event.ts`.

8. **El replay visual/local del CLI es otro problema distinto.** `replay-time-order` corrigió la mezcla entre filas locales no persistidas y mensajes persistidos para no inferir cronología a partir de IDs.  
   Commit: `642d6c52c4ec43192c75c1adbd9df18554087218`.

9. **Los drafts inspeccionados no son estado durable del backend.** `session-prompt-drafts` guarda drafts por Session en un `Map` del cliente TUI.  
   Commit: `ea74a84ea37df6ee75789e27ebda1ec5b25ab7f3`.

10. **La arquitectura actual es híbrida durante la transición V1/V2.** El mismo flujo durable puede alimentar proyecciones de compatibilidad `session/message/part` y una proyección secuenciada `session_message`/`session_input`.  
    Referencias: `packages/core/src/session/projector.ts`, `packages/core/src/session/sql.ts`.

## Evolución funcional observada

La secuencia de branches/commits muestra una convergencia hacia cuatro boundaries claros:

- **Event log durable** como fuente de orden, replay y sincronización.
- **Projectors** como materialización transaccional de vistas de lectura.
- **SessionStore** como read-side explícito sobre proyecciones.
- **State machine de ejecución** separada del estado durable de la conversación.

Ejemplos de hitos:

- `storage-v2-service`: consolidación del servicio SQLite/Effect para V2.
- `refactor/core-v2-message-identity`: separación de identidad message/event y reset controlado del estado beta reconstruible.
- `protocol-events` / `published-events`: estabilización del boundary de eventos publicados.
- `session-event-stream`: exposición de historia + stream durable por Session.
- `step-settlement`: clasificación única del resultado terminal de un intento.
- `step-machine`: extracción de retry/compaction/continuation a una máquina de estados explícita.
- `message-row-codec`: codec centralizado entre modelo de mensaje y fila SQLite.
- `session-store-reads`: extracción de lecturas proyectadas hacia `SessionStore`.

## Índice

1. [Inventario y agrupación de branches](01-inventario-branches.md)
2. [Modelo de Session y Message](02-modelo-session-message.md)
3. [State machine, steps y lifecycle de ejecución](03-state-machine-steps.md)
4. [Eventos, streaming, ordering y versionado](04-eventos-streaming-ordering.md)
5. [Persistencia, SQLite y migraciones](05-persistencia-sqlite-migraciones.md)
6. [Replay, resume, recovery, fork y revert](06-replay-resume-recovery-forks.md)
7. [Sincronización backend/clientes](07-sincronizacion-clientes.md)

## Paths de referencia principales en `dev`

- `packages/opencode/src/session/session.ts`
- `packages/opencode/src/session/message-v2.ts`
- `packages/opencode/src/session/processor.ts`
- `packages/opencode/src/session/run-state.ts`
- `packages/opencode/src/session/status.ts`
- `packages/opencode/src/id/id.ts`
- `packages/opencode/src/event-v2-bridge.ts`
- `packages/core/src/session.ts`
- `packages/core/src/session/sql.ts`
- `packages/core/src/session/projector.ts`
- `packages/core/src/event.ts`
- `packages/core/src/event/sql.ts`
- `packages/core/src/database/migration.ts`
- `packages/schema/src/session-event.ts`

## Límite del análisis

Las branches de tabs, picker, scroll, layout y otros cambios puramente visuales se inventarían y clasifican, pero no se usan para deducir el state machine del backend salvo que cambien contratos de Session/Event. Esto evita confundir estado de presentación con estado durable o de ejecución.