# Eventos, streaming, ordering y versionado

## 1. Event surface

La branch `protocol-events` contiene el commit `6ad5ce39beb88ababd7d429e45769ea61b2795d2`, `refactor(protocol): define current event surface`, seguido por tests del boundary de grupo y por `refactor(schema): share server event assembly`.

**Hecho confirmado:** la superficie de eventos dejó de ser una colección accidental de tipos internos y pasó a tratarse como contrato de protocolo agrupado.

La branch `published-events` contiene `95f264e04eca61b4a81ef92ee29476556d299a68`, `refactor(schema): distinguish published event durability`, y commits que prueban publication branches y preservan routing semantics.

**Hecho confirmado:** “publicado” y “durable” son propiedades distintas. Un evento puede existir para distribución live sin formar parte del historial durable replayable.

## 2. Durable EventV2

`packages/core/src/event.ts` implementa el servicio durable. Un event definition durable declara:

- aggregate key, para Session normalmente `sessionID`;
- version;
- schema de payload.

El tipo almacenado incorpora versión, por ejemplo `session.next.step.ended.2`.

Las tablas están en `packages/core/src/event/sql.ts`:

- `event`
- `event_sequence`

## 3. Ordering

El orden autoritativo de eventos de una Session es:

```text
aggregate_id = sessionID
seq = 1, 2, 3, ...
```

No se define por:

- timestamp del payload;
- timestamp SQLite;
- Event ID;
- Message ID;
- orden de llegada a un cliente.

`event_sequence` mantiene el cursor/owner del aggregate; `event` conserva cada evento con su `seq`.

## 4. Commit transaccional y projectors

El flujo durable ejecuta en una transacción SQLite:

1. resolver/validar sequence;
2. validar replay/idempotencia si procede;
3. ejecutar projectors registrados;
4. actualizar sequence;
5. insertar el evento durable;
6. completar la transacción;
7. distribuir el hecho hacia superficies live/publicadas según el routing.

**Hecho confirmado:** proyección y registro durable están dentro del mismo transaction boundary.

**Inferencia:** las materialized views no necesitan un reconciler eventual para el caso normal local; el diseño busca consistencia fuerte entre log y proyección en cada commit.

## 5. Replay durable e idempotencia

Cuando se importa/reproduce un evento con `seq` ya observado, el core comprueba la fila existente.

Sólo considera idempotente el replay si coinciden identidad, tipo y datos. Si el mismo sequence representa otro hecho, produce una divergencia de replay.

También se rechazan gaps incompatibles con el sequence esperado.

Esto convierte `seq` en un mecanismo de detección de split-brain/divergencia, no sólo en un contador para ordenar UI.

## 6. Eventos live-only

`packages/schema/src/session-event.ts` marca explícitamente varios fragmentos de stream como live-only. Ejemplos confirmados en el schema:

- `Text.Delta`
- `Reasoning.Delta`

Sus correspondientes `Ended` contienen el valor completo replayable.

Patrón:

```text
Text.Started   durable
Text.Delta     live-only, 0..N
Text.Ended     durable con texto completo
```

**Inferencia:** se evita inflar el log durable con cada token/chunk sin perder capacidad de reconstruir el estado final.

## 7. Session event lifecycle

Un flujo típico puede materializar:

```text
Prompted / PromptAdmitted
Step.Started
Reasoning.Started
Reasoning.Delta ...
Reasoning.Ended
Text.Started
Text.Delta ...
Text.Ended
Tool.* ...
Step.Ended | Step.Failed
```

No todos los paths contienen todos los eventos; tools, shell, compaction, switches y synthetic/context events amplían la máquina.

## 8. `session-event-stream`

La branch contiene, entre otros, el commit `5146f01e0a8826318056fdb38477fb0202c29099`, `fix(sdk): preserve session stream ordering`.

**Hecho confirmado:** el stream expuesto a SDK tuvo que preservar explícitamente el orden de Session; el ordering no se puede delegar a timing de red o a concurrencia de handlers.

La propia branch tuvo conflictos en `session/execution`, `run-coordinator` y el grupo de protocolo Session al integrarse con `dev`, lo que demuestra que el stream cruzaba boundaries runtime/protocolo y no era una feature únicamente de UI.

## 9. `published-events`

La distinción introducida por esta línea puede modelarse como:

```text
internal event
   |
   +-- published/live surface
   |
   +-- durable? -> event log + replay/sync
```

Los commits `45071a8d8e293b26284ac14bd21a248fa0616c99` y `603b334b7f70620e8e531fadcd1f0ff159f186ab` prueban branches de publicación y preservan routing semantics.

**Inferencia:** OpenCode necesitaba evitar dos errores opuestos: persistir ruido puramente live y ocultar eventos que sí debían cruzar el boundary servidor/cliente.

## 10. Versionado y resets beta

`normalize-step-event-versions`, commit `179a8748cb9ba02cf6a633ed28da8c4fd0b4e9c0`, reseteó eventos experimentales después de descartar historia beta. En ese momento Step settlement volvió a `.1`; el `dev` actual usa versión 2 para ese payload.

### Regla de interpretación

Nunca inferir compatibilidad sólo por comparar números de versión entre branches históricas. Hay que comprobar:

- si la historia era publicada o experimental;
- qué tablas se resetearon;
- qué decoder existía;
- si la versión anterior llegó a considerarse estable.

## 11. Event batching

Las branches `event-batching` y equivalentes exploran reducción del overhead de publicación/sync mediante grupos de eventos. Su existencia es coherente con la separación durable/publicada: batching es una política de transporte/entrega y no debe cambiar el `seq` lógico del aggregate.

**Inferencia:** cualquier batching correcto debe preservar ordering e identidad individual aunque cambie la granularidad de transporte.

## 12. Conclusiones

### Confirmado

- existe un catálogo/protocolo explícito de eventos;
- published y durable son dimensiones distintas;
- durable Session events se ordenan por aggregate `seq`;
- projectors participan en el mismo transaction boundary;
- deltas de streaming pueden ser live-only;
- el SDK/session stream tiene requisitos explícitos de ordering;
- event types durables están versionados.

### Inferencia

- el event system es simultáneamente mecanismo de persistence V2, integración y sync;
- la separación live/durable minimiza coste sin sacrificar replay;
- versioning y sequence constituyen el protocolo de consistencia entre productores, almacenamiento y consumidores.