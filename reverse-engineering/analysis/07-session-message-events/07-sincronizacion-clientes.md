# Sincronización entre backend y clientes

## Baseline auditada

Contrastado contra `dev@dc4449df0d52199704ea4989a5a993ebbc605612`.

## 1. No existe un único mecanismo de sync

En `dev` convergen:

- APIs de estado materializado;
- proyecciones SQLite;
- event stream global publicado;
- history/stream durable por Session;
- `EventV2Bridge` para surfaces `packages/opencode`;
- SDK/protocol contracts.

La estrategia permite bootstrap desde estado actual y seguimiento incremental, pero cada surface tiene ordering y durability propios.

## 2. Bootstrap desde read models

`SessionV2.Service` puede listar Sessions y mensajes desde las proyecciones SQLite. `SessionStore` es una dependencia read-side más estrecha para get/context/runnerContext/message, no el dueño actual de todas las queries paginadas.

Un cliente no necesita replayar el event log completo desde el origen para obtener una vista inicial.

## 3. Ordering: varias unidades distintas

### Session list V2

`SessionV2.list()` ordena actualmente por:

```text
time_created + Session ID tie-breaker
```

con anchors y dirección previous/next.

### Session list legacy-compatible

`packages/opencode/src/session/session.ts` posee sus propias queries y en algunas surfaces ordena por `time_updated`. No debe trasladarse ese criterio al list V2.

### Messages V2

`SessionV2.messages()` pagina por `SessionMessageTable.seq`, derivado del durable event sequence.

### Event history

`SessionV2.history()` / `EventV2.readAggregate()` ordenan por aggregate `seq` ascendente después del cursor `after`.

### Live chunks

El stream live conserva el orden de entrega durante una conexión, pero sus deltas no son necesariamente durable/replayable.

Por tanto, no existe un “orden por ID” ni un “orden por updated time” universal.

## 4. Global event stream

`packages/server/src/handlers/event.ts` ofrece SSE global:

- queue bounded 256;
- `server.connected` inicial;
- heartbeat 15 s;
- stream de `EventV2.allBounded()`.

Es una superficie live/publicada y no debe confundirse con history durable de una Session concreta.

## 5. Session history y event stream durable

`packages/server/src/handlers/session.ts` expone handlers V2 para:

- `session.history`: página de durable events con `after`/`limit` y `hasMore`;
- `session.events`: stream de eventos de la Session después de `after`.

Internamente `SessionV2.history()` usa `EventV2.readAggregate()` y `SessionV2.events()` usa `EventV2.durable(...)` filtrado por `SessionEvent.Durable`.

Esto proporciona una surface explícita de replay/follow por aggregate sequence.

## 6. Published no implica durable

Algunos eventos/deltas existen para progreso live y no reaparecen como entradas equivalentes en history. Los boundaries terminales/durable permiten reconstruir estado final sin exigir haber observado cada chunk.

Un reconnect correcto no debe depender de que el cliente haya recibido todos los deltas previos.

## 7. EventV2Bridge

`packages/opencode/src/event-v2-bridge.ts` adapta EventV2 al `GlobalBus` usado por consumers legacy-compatible.

Para cada evento emite una notificación normal. Si el evento es durable, emite además:

```text
{
  type: "sync",
  syncEvent: {
    id,
    type: versionedType(...),
    seq,
    aggregateID,
    data
  }
}
```

Esto permite propagar facts durable sin exigir que todos los consumidores internos migren simultáneamente a la API Core.

## 8. SDK/protocol

Los groups de protocol y handlers server convierten estas surfaces en contratos consumibles por clientes generados. Los cambios de schema/version de eventos pueden por tanto afectar al SDK.

Branches como `protocol-events`, `published-events` y `session-event-stream` son evidencia evolutiva, pero el contrato vigente debe leerse siempre del source `dev`.

## 9. Reconnect: patrón seguro

La arquitectura soporta un patrón conceptual:

```text
1. leer snapshot/proyección actual
2. conocer un boundary durable/cursor
3. leer history posterior si hace falta
4. seguir stream live/durable
5. deduplicar/aplicar según identidad y sequence
```

La API concreta puede encapsular parte de estos pasos. No se afirma que todos los clientes implementen exactamente este algoritmo.

## 10. Duplicate delivery y divergencia

EventV2 proporciona idempotencia estricta para replay durable en el store: mismo `(aggregate, seq)` sólo puede representar el mismo id/type/data.

Eso protege el historial persistido, pero un cliente que consuma streams publicados todavía debe diseñar sus propios reducers/dedup de forma adecuada; la garantía del storage no convierte automáticamente cualquier transport consumer en idempotente.

## 11. Estado que no debe sincronizarse como dominio durable

Ejemplos de estado local/transitorio:

- draft aún no admitido;
- scroll/layout/picker/tab state;
- ciertos deltas una vez consolidado el boundary final;
- estado runtime como busy/retry/idle cuando no existe como fact durable equivalente.

Esto debe distinguirse de Session metadata, mensajes, durable events, model/agent switches, prompt admission y outcomes de steps/tools.

## Arquitectura resumida

```text
SQLite projections ---------> bootstrap/read
       ^                            |
       |                            v
Durable EventV2 <------ Session API / protocol ------> clients
       |                            |
       | history + session stream   | global/live SSE
       +----------------------------+

EventV2Bridge -> GlobalBus consumers legacy-compatible
```

## Invariantes confirmados

1. read models y event surfaces coexisten;
2. `SessionV2.list()` y legacy list no comparten necesariamente sort key;
3. messages/history V2 usan sequence como ordering durable;
4. global SSE y Session durable stream son superficies diferentes;
5. published/live no equivale a durable/replayable;
6. EventV2Bridge propaga durable metadata hacia el bus de compatibilidad;
7. SDK/protocol es parte del boundary de sync, no sólo la UI.

## Referencias

- `packages/core/src/session.ts`
- `packages/core/src/session/store.ts`
- `packages/core/src/event.ts`
- `packages/opencode/src/session/session.ts`
- `packages/opencode/src/event-v2-bridge.ts`
- `packages/server/src/handlers/event.ts`
- `packages/server/src/handlers/session.ts`
