# 07 — Sessions, mensajes, eventos y persistencia

## Objetivo

Este dossier reconstruye Session, Message, state/steps, event streams, persistence, replay, resume, recovery y sincronización cliente/backend tomando `dev@dc4449df0d52199704ea4989a5a993ebbc605612` como baseline.

## Regla de lectura

En `dev` conviven dos generaciones:

- surface/runtime legacy-compatible en `packages/opencode`, todavía activa;
- modelo/event/session V2 en `packages/core`, también real y consumido por las APIs V2.

No se colapsan ambos modelos en una única máquina ficticia.

## Arquitectura resumida

```text
product/LLM/session execution
          |
          +--> legacy-compatible Session Message/Part events
          +--> V2 durable Session events
                         |
                         v
                    EventV2 publish
              [SQLite transaction]
                         |
          +--------------+----------------+
          |                               |
     projectors                      event/event_sequence
          |
          +--> session/message/part
          +--> session_message/session_input
                         |
                         v
             Session / SessionV2 read APIs
                         |
          +--------------+----------------+
          |                               |
   global published events        per-Session history/events
          |                               |
          +--------------+----------------+
                         v
                      clients
```

## Invariantes principales auditados

1. **El ordering durable por aggregate se define por `seq`.** `EventV2.latestSequence()` parte de `-1`, de modo que un aggregate nuevo recibe primero `seq = 0` y luego incrementa secuencialmente.
2. **Published/live no equivale a durable.** Deltas de streaming pueden existir sólo en la surface live mientras boundaries completos quedan replayables.
3. **El commit durable y los projectors registrados comparten una transacción SQLite.** La implementación está dentro de `EventV2.publish()`/`commitDurableEvent()`; no existe una API pública denominada literalmente `EventV2.commit()`.
4. **Replay es estricto.** Un evento con `seq <= latest` sólo es idempotente si coinciden ID, type persistido/versionado y data.
5. **`parentID` de Session y fork son conceptos distintos.** Child session representa relación estructural; `Session.fork()` legacy-compatible crea otra Session y remapea mensajes/parts.
6. **`busy/retry/idle` es estado operativo/transitorio.** No sustituye al event history durable.
7. **Message ID y Event ID son dominios distintos.** El modelo V2 usa identidades separadas.
8. **V1 y V2 comparten/proyectan estado durante la migración.** `SessionProjector` actualiza tanto vistas legacy-compatible como read models V2 según el evento.
9. **La tabla `event` vigente no almacena timestamp de creación.** Su ordering se apoya en aggregate + sequence.
10. **Los read paths tienen distintos orderings.** `SessionV2.list()` usa `time_created` + ID; mensajes/history V2 usan sequence; surfaces legacy pueden usar otros criterios.

## Documentos

1. [Inventario y agrupación de branches](01-inventario-branches.md)
2. [Modelo de Session y Message](02-modelo-session-message.md)
3. [State machine, steps y lifecycle de ejecución](03-state-machine-steps.md)
4. [Eventos, streaming, ordering y versionado](04-eventos-streaming-ordering.md)
5. [Persistencia, SQLite y migraciones](05-persistencia-sqlite-migraciones.md)
6. [Replay, resume, recovery, fork y revert](06-replay-resume-recovery-forks.md)
7. [Sincronización backend/clientes](07-sincronizacion-clientes.md)

## Paths primarios

- `packages/opencode/src/session/session.ts`
- `packages/opencode/src/session/message-v2.ts`
- `packages/opencode/src/session/processor.ts`
- `packages/opencode/src/session/run-state.ts`
- `packages/opencode/src/session/status.ts`
- `packages/opencode/src/event-v2-bridge.ts`
- `packages/core/src/session.ts`
- `packages/core/src/session/store.ts`
- `packages/core/src/session/sql.ts`
- `packages/core/src/session/projector.ts`
- `packages/core/src/event.ts`
- `packages/core/src/event/sql.ts`
- `packages/server/src/handlers/event.ts`
- `packages/server/src/handlers/session.ts`
- `packages/schema/src/session-event.ts`

## Convención epistemológica

- **Hecho confirmado:** observable directamente en source/schema/test/diff concreto.
- **Histórico:** perteneciente a una branch/commit y no asumido en `dev`.
- **Inferencia:** interpretación arquitectónica derivada de varios hechos.

El nombre de una branch nunca se considera suficiente por sí solo para declarar comportamiento vigente.

## Conclusión

La arquitectura de Session está en migración, pero la coexistencia no es ambigua si se distinguen write/event facts, read projections y runtime ownership. El error principal que esta auditoría evita es proyectar una característica de V2 sobre todas las surfaces legacy-compatible —o viceversa— sin verificar el composition path que realmente la consume.