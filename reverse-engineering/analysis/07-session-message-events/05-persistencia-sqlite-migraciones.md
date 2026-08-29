# Persistencia, SQLite y migraciones

## 1. Boundary de persistencia

La arquitectura actual usa SQLite como store local transaccional para Sessions, mensajes, eventos y proyecciones. Hay que distinguir:

- tablas canónicas/históricas V1: `session`, `message`, `part`;
- log durable V2: `event`, `event_sequence`;
- proyecciones/admission V2: `session_message`, `session_input` y tablas relacionadas.

Paths clave:

- `packages/core/src/event/sql.ts`
- `packages/core/src/session/sql.ts`
- `packages/core/src/database/migration.ts`
- `packages/core/src/database/sqlite.ts`
- `packages/core/src/session/projector.ts`

## 2. Event tables

`event` conserva el hecho durable, su tipo versionado, payload, aggregate ID, sequence y creación.

`event_sequence` mantiene el último `seq` por aggregate y metadata de ownership usada para coordinación/consistencia.

**Hecho confirmado:** el sequence no se deriva recalculando `MAX(seq)` en cada publicación; existe estado explícito del aggregate.

## 3. Proyecciones de Session

`packages/core/src/session/projector.ts` aplica eventos de Session a vistas de lectura.

Durante la transición se mantienen proyecciones paralelas:

- compatibilidad V1 sobre `session/message/part`;
- timeline V2 sobre `session_message`.

**Hecho confirmado:** el mismo evento durable puede alimentar más de una representación.

**Inferencia:** los projectors encapsulan compatibilidad y permiten sustituir gradualmente el read model sin exigir una migración big-bang de todos los clientes.

## 4. Transaction boundary

`EventV2.commit()` ejecuta projectors y persistencia del evento dentro de la misma transacción SQLite.

Consecuencia:

```text
no existe estado normal:
  evento committed
  pero projector principal no aplicado
```

Si falla un projector, falla el commit completo.

**Inferencia:** el sistema prefiere consistencia síncrona local frente a proyecciones eventually-consistent para el timeline principal.

## 5. `SessionStore` como read side

`session-store-reads`, commit `bc922b8ac590a48ae0fa4b14919c04f2886022d5`, extrae consultas proyectadas hacia `SessionStore`.

Incluye:

- list de Sessions con filtros y anchor;
- messages paginados por `session_message.seq`;
- decodificación de filas mediante history/codec.

### Ordering de Session list

El list usa `time_updated` y un ID tie-breaker para obtener paginación estable.

### Ordering de messages

Messages usa `seq` como boundary del cursor.

**Inferencia:** esta extracción formaliza un patrón CQRS parcial: Session service gobierna commands/policy y Store gobierna materialized reads.

## 6. `message-row-codec`

Commit `ffaaad89a44fd4069c8f5f232c9cfaea1163dc6b` centraliza transformación Message ↔ row.

Esto es relevante para persistence porque elimina múltiples interpretaciones del JSON `data` y establece que columnas canónicas como `id` y `type` prevalecen.

## 7. `storage-v2-service`

La línea contiene commits como:

- `b3d6f931484137d271039272a9cc756c741f9095` — compartir Effect SQLite database layer;
- `71423b9a5825eed2a018b5cc118dcf26f19ed6f1` — mantener setup de DB de tests explícito;
- `178840489f3c99b1a47f1aa94ca25585c8dd1e03` — documentar cleanup/bootstrap;
- `c633d10e744f1977c1776cc040eadf6772426950` — simplificar storage contract setup.

**Hecho confirmado:** el acceso SQLite se convirtió en un service/layer compartido y testeable dentro de la arquitectura Effect.

**Inferencia:** persistence deja de pertenecer implícitamente al módulo Session y pasa a ser una dependency injectable del core.

## 8. Branch `sqlite`

`sqlite` es una línea mucho más antigua, con commits amplios de sincronización. Su valor es arqueológico: demuestra la adopción temprana de SQLite, pero no permite atribuir finamente los invariantes actuales a un único diff.

Se considera antecedente, no baseline.

## 9. Migraciones

`packages/core/src/database/migration.ts` mantiene una tabla de migraciones y ejecuta cambios ordenados de schema/data.

Las migraciones son especialmente importantes durante V2 porque algunas tablas eran declaradas explícitamente experimentales y reconstruibles.

## 10. Reset boundary de V2 beta

El commit `f2d133edf7430f3229a84c29d6b0e1690f1f2a60` añadió una migración que elimina contenido de:

- `session_input`;
- `session_message`;
- `event`;
- `event_sequence`.

Y documenta que no deben truncarse como parte de ese reset:

- `session`;
- `message`;
- `part`.

### Interpretación

**Hecho confirmado:** en ese momento la historia V1 era la frontera de compatibilidad preservada y el estado V2 aún podía reconstruirse/descartarse.

**Hecho confirmado:** resetear `event`/`event_sequence` eliminaba capacidad de replay/warp V2 previa hasta generar nueva historia durable.

## 11. Schema evolution de eventos

`normalize-step-event-versions` documenta otro reset de estado experimental para cambiar contratos incompatibles sin cargar decoders eternos durante la beta.

Esto ilustra dos estrategias:

1. migración compatible cuando historia estable debe sobrevivir;
2. reset explícito cuando el estado sigue siendo beta y está declarado disposable.

## 12. IDs persistidos

Los IDs modernos están tipados por dominio. El caso más importante es:

```text
Event.ID          evt_*
SessionMessage.ID msg_*
```

La persistencia ya no usa implícitamente el mismo string como identidad de ambos objetos.

**Inferencia:** prefijos y branded schemas actúan como defensa runtime/type-level contra joins y lookups en el dominio equivocado.

## 13. Recovery y durability

La persistencia permite que una Session siga siendo legible aunque su directory/worktree ya no exista. `orphan-session-recovery` demuestra que location availability y durability de la historia son concerns distintos.

La eliminación física de un worktree no equivale a borrar Session/event history.

## 14. Invariantes de persistence

### Confirmados

- SQLite es el store transaccional principal local.
- event sequence se persiste por aggregate.
- projectors y event commit comparten transacción.
- V2 tiene read model `session_message` ordenado por `seq`.
- hay migrations explícitas y una frontera documentada entre estado beta descartable y V1 preservado.
- codecs encapsulan representación de Message rows.

### Inferencias

- el diseño busca separar write model/event log de read models sin introducir consistencia eventual local.
- los resets beta permitieron iterar schemas rápidamente antes de declarar compatibilidad estable.
- `SessionStore` es el inicio de un read-side más autónomo y potencialmente reemplazable.