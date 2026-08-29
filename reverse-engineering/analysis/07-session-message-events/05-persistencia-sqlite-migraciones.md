# Persistencia, SQLite y migraciones

## Baseline auditada

Contrastado contra `dev@dc4449df0d52199704ea4989a5a993ebbc605612`.

## 1. Boundary de persistencia

OpenCode usa SQLite como store local transaccional para Sessions, mensajes, eventos y proyecciones. En `dev` conviven varias representaciones:

- compatibilidad V1: `session`, `message`, `part`;
- log durable: `event`, `event_sequence`;
- read/admission V2: `session_message`, `session_input` y tablas relacionadas.

Paths de mayor peso:

- `packages/core/src/event/sql.ts`
- `packages/core/src/session/sql.ts`
- `packages/core/src/event.ts`
- `packages/core/src/session/projector.ts`
- `packages/core/src/session/store.ts`
- `packages/core/src/session.ts`
- `packages/core/src/database/migration.ts`

## 2. Tablas del event log

`EventSequenceTable` contiene exactamente:

- `aggregate_id` primary key;
- `seq`;
- `owner_id` opcional.

`EventTable` contiene exactamente:

- `id` primary key;
- `aggregate_id` FK a `event_sequence`;
- `seq`;
- `type`;
- `data` JSON.

**Corrección:** la tabla `event` actual **no tiene columna de creación/timestamp**. El orden durable procede de `(aggregate_id, seq)`.

Indices confirmados:

- unique `(aggregate_id, seq)`;
- `(aggregate_id, type, seq)`.

## 3. Sequence explícito

`event_sequence` mantiene el último sequence por aggregate; EventV2 no necesita recalcular `MAX(seq)` para cada publicación.

Un aggregate nuevo parte conceptualmente de `latest = -1`, por lo que su primer evento recibe `seq = 0`.

## 4. Projectors y transacción

`packages/core/src/session/projector.ts` registra projectors sobre EventV2.

El mismo componente mantiene durante la transición:

- filas V1 `session/message/part`;
- `session_message` V2;
- admission/state en `session_input`;
- campos derivados de Session, usage y cambios agent/model/location.

Los projectors se ejecutan dentro de la misma transacción que avanza `event_sequence` e inserta el evento durable.

**Hecho confirmado:** un fallo del projector aborta ese commit durable local.

## 5. Ordering de `session_message`

Cuando el projector inserta un `SessionMessage`, usa:

```text
session_message.seq = event.durable.seq
```

Así el read model V2 hereda el ordering del event aggregate.

`SessionV2.messages()` pagina por ese `seq`; si se suministra un cursor de Message, primero obtiene el sequence de la fila anchor y consulta antes/después de ese boundary.

## 6. `SessionStore` actual

`packages/core/src/session/store.ts` **no expone actualmente `list()` ni paginación de mensajes**. Su interfaz vigente contiene:

- `get(sessionID)`;
- `context(sessionID)`;
- `runnerContext(sessionID, baselineSeq)`;
- `message(messageID)`.

Las operaciones de list/messages paginadas están implementadas en `SessionV2.Service` (`packages/core/src/session.ts`) sobre las mismas tablas.

Esta distinción corrige una atribución anterior basada en la branch histórica `session-store-reads`.

## 7. Ordering de Session list V2

`SessionV2.list()` usa actualmente como sort column:

```text
SessionTable.time_created
```

con `SessionTable.id` como tie-breaker, y soporta `asc/desc` + anchors previous/next.

**Corrección:** no debe afirmarse que el list V2 actual ordena por `time_updated`; ese criterio aparece en otras/antiguas surfaces, incluido el list legacy de `packages/opencode`, pero no en esta implementación V2.

## 8. Compatibilidad V1

`packages/opencode/src/session/session.ts` sigue leyendo/escribiendo el `SessionTable` compartido mediante APIs legacy-compatible, y `MessageV2` trabaja sobre `MessageTable`/`PartTable`.

La coexistencia de esas tablas con `session_message` no implica que una de las representaciones pueda eliminarse hoy sin revisar consumidores.

## 9. Codec y autoridad de columnas

La evolución `message-row-codec` formaliza que identity/discriminant estructurales deben venir de columnas canónicas y no de copias stale dentro del JSON `data`.

El source actual de `SessionProjector` codifica `SessionMessage`, separa `id` y `type`, y guarda el resto en `data`.

## 10. Migraciones y resets beta

`packages/core/src/database/migration.ts` contiene migraciones ordenadas de schema/data.

Durante la evolución V2 hubo resets explícitos de tablas reconstruibles/experimentales, incluidos `session_input`, `session_message`, `event` y `event_sequence`, mientras se preservaban superficies V1 en migraciones concretas.

Esos resets son hechos históricos de determinadas migraciones; no significan que las tablas V2 actuales deban considerarse descartables permanentemente.

## 11. IDs de dominio

El diseño moderno distingue namespaces de identidad, por ejemplo:

```text
Event.ID           evt_*
SessionMessage.ID  msg_*
```

La separación evita tratar un evento y su proyección de dominio como la misma entidad por accidente.

## 12. Durability frente a filesystem/location

La historia de Session reside en SQLite/event projections; la disponibilidad física del worktree/location es otro concern. Recovery/move pueden operar sobre Sessions persistidas aunque el filesystem original ya no esté disponible.

## Invariantes confirmados

1. SQLite es el store transaccional local principal para estas superficies.
2. `event` no guarda timestamp de creación en el schema vigente.
3. el sequence durable se persiste explícitamente por aggregate.
4. projectors y event insertion forman un transaction boundary.
5. `session_message.seq` deriva del durable event sequence.
6. `SessionStore` actual es un read service estrecho; list/messages pagination están en `SessionV2.Service`.
7. `SessionV2.list()` ordena por `time_created` + ID tie-breaker en `dev`.
8. V1 y V2 siguen coexistiendo sobre SQLite.

## Referencias

- `packages/core/src/event/sql.ts`
- `packages/core/src/event.ts`
- `packages/core/src/session/sql.ts`
- `packages/core/src/session/projector.ts`
- `packages/core/src/session/store.ts`
- `packages/core/src/session.ts`
- `packages/opencode/src/session/session.ts`
