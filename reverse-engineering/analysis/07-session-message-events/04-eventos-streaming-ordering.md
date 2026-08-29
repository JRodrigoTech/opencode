# Eventos, streaming, ordering y versionado

## Baseline auditada

Contrastado contra `dev@dc4449df0d52199704ea4989a5a993ebbc605612`.

## 1. Published y durable son dimensiones distintas

El event system no convierte automáticamente toda notificación en historia replayable. `EventV2` soporta eventos live/no durable y definitions durables bajo la misma superficie.

Un evento durable publicado adquiere:

```text
durable.aggregateID
durable.seq
durable.version
```

Para los eventos de Session, el aggregate habitual es `sessionID`.

## 2. Orden durable

`packages/core/src/event.ts` usa `event_sequence` como cursor por aggregate.

`latestSequence()` devuelve `-1` cuando todavía no existe sequence. Por tanto, con la implementación actual, **el primer evento durable nuevo recibe `seq = 0`**, y los siguientes avanzan `1, 2, ...`.

La propiedad importante no es el valor inicial sino el invariante:

```text
next seq = latest + 1
```

No debe documentarse el orden durable como timestamp, Event ID, Message ID ni orden de llegada a un cliente.

## 3. Commit transaccional

Para un evento durable nuevo, `commitDurableEvent()` ejecuta en una transacción SQLite `immediate`:

1. lee `event_sequence` y ownership;
2. valida aggregate/sequence/replay;
3. ejecuta projectors registrados para el tipo lógico;
4. ejecuta el hook local `PublishOptions.commit` si existe;
5. actualiza `event_sequence`;
6. inserta la fila en `event`.

Sólo después del commit se despiertan subscribers durable y se distribuye el evento publicado.

**Hecho confirmado:** los projectors registrados y el event insert comparten el mismo transaction boundary local.

## 4. Replay e idempotencia

Si un replay llega con `seq <= latest`, EventV2 busca la fila existente en ese aggregate/sequence.

Sólo se considera idempotente cuando coinciden:

- ID;
- type persistido/versionado;
- data codificada.

Si no coinciden se produce `InvalidDurableEventError` por divergencia.

También se rechazan gaps de sequence, event IDs reutilizados en otra posición, aggregate mismatch y conflictos de owner cuando la política correspondiente aplica.

## 5. Lectura durable

`EventV2.readAggregate()` consulta eventos con:

```text
aggregate_id = input.aggregateID
seq > after
ORDER BY seq ASC
LIMIT limit + 1
```

El elemento adicional permite calcular `hasMore`.

`SessionV2.history()` usa este mecanismo con el manifest `SessionDurable`.

## 6. Stream durable de Session

`SessionV2.events({ sessionID, after })` se construye sobre:

```text
EventV2.durable({ aggregateID: sessionID, after })
```

y filtra los eventos por `SessionEvent.Durable`.

Esto es distinto del stream global: la superficie de Session está anclada a un aggregate y a su sequence.

## 7. Stream global SSE

`packages/server/src/handlers/event.ts` implementa un stream SSE global con:

- subscriber capacity `256`;
- `EventV2.allBounded()`;
- evento sintético inicial `server.connected`;
- heartbeat cada `15 seconds`;
- headers anti-cache/anti-buffer.

Cuando el queue bounded no puede aceptar más elementos, el subscriber puede fallar con `SubscriberOverflowError` en vez de consumir memoria sin límite.

## 8. Live-only frente a durable

El schema de Session distingue boundaries completos de deltas de streaming. Algunos deltas son live-only mientras eventos terminales conservan el valor replayable.

Por tanto:

```text
haberlo visto en live stream != existir necesariamente en durable history
```

Un cliente que se reconecta debe reconstruir desde historia/proyecciones durable y no asumir que recuperará cada chunk intermedio.

## 9. Versionado

Los eventos durables se almacenan mediante:

```text
versionedType(definition.type, durable.version)
```

`decodeSerializedEvent()` exige que ese type esté registrado en el durable manifest y decodifica el payload con el schema correspondiente.

Existe además un TODO explícito: los projectors todavía no están ligados a type+version exactos para soportar payloads históricos incompatibles arbitrarios. Por ello no debe afirmarse compatibilidad retroactiva ilimitada entre todas las versiones.

## 10. EventV2Bridge

`packages/opencode/src/event-v2-bridge.ts` adapta EventV2 al ecosistema `packages/opencode`.

Al publicar, completa `location` desde `InstanceRef`/`WorkspaceRef` cuando el caller no la suministra.

Al escuchar, emite hacia `GlobalBus`:

1. un evento normal con `id`, `type` y `properties`;
2. para eventos durables, un segundo payload `type: "sync"` con:
   - `id`;
   - type versionado;
   - `seq`;
   - `aggregateID`;
   - `data`.

Este bridge es un anti-corruption layer real entre el event core y surfaces legacy-compatible.

## 11. Branches históricas

Branches como `protocol-events`, `published-events`, `session-event-stream` y `normalize-step-event-versions` sirven para reconstruir la evolución del contrato, pero sus números de versión, ejemplos de type o mecanismos intermedios no deben sustituir el source de `dev`.

## Invariantes confirmados

1. el sequence durable es por aggregate y empieza en 0 en un aggregate nuevo con la implementación actual;
2. `seq` es el orden autoritativo del event history;
3. projectors y event commit comparten transacción;
4. replay repetido sólo es idempotente si representa exactamente el mismo hecho;
5. global SSE y durable Session stream son superficies distintas;
6. published/live no implica durable/replayable;
7. durable types están versionados, pero no existe una promesa de compatibilidad arbitraria entre schemas históricos.

## Referencias

- `packages/core/src/event.ts`
- `packages/core/src/event/sql.ts`
- `packages/core/src/session.ts`
- `packages/opencode/src/event-v2-bridge.ts`
- `packages/server/src/handlers/event.ts`
- `packages/schema/src/session-event.ts`
