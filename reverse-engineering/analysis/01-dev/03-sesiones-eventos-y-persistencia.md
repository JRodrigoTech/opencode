# 03 — Sesiones, eventos y persistencia

## La sesión como agregado operativo

Paths principales:

- `packages/opencode/src/session/session.ts`
- `packages/opencode/src/session/message.ts`
- `packages/opencode/src/session/message-v2.ts`
- `packages/opencode/src/session/status.ts`
- `packages/opencode/src/session/run-state.ts`
- `packages/core/src/session/sql.ts`

**[CONFIRMADO]** Una sesión contiene identidad, relación con proyecto/workspace/directorio, metadatos de agente/modelo, timestamps y relación opcional parent/child. Los mensajes se descomponen en parts que permiten representar texto, reasoning, herramientas y otros fragmentos estructurados.

**[CONFIRMADO]** El runtime no mantiene la conversación sólo en memoria: los cambios relevantes se escriben durante la ejecución y generan señales/eventos consumibles por otros subsistemas.

## Dos generaciones coexistiendo

`dev` contiene simultáneamente:

- `packages/opencode/src/session/session.ts` y contratos/eventos V1;
- `packages/core/src/v1/session.ts` como compatibilidad V1;
- modelos nuevos en `packages/core/src/session/*`;
- `packages/opencode/src/event-v2-bridge.ts`;
- `packages/core/src/event.ts` (`EventV2`);
- `packages/core/src/session/projector.ts`.

**[CONFIRMADO]** La composición del server provee tanto servicios V1 como V2. La coexistencia es funcional, no meramente código muerto.

## Modelo SQL

`packages/core/src/database/database.ts` y `packages/core/src/session/sql.ts` son las fuentes principales.

**[CONFIRMADO]** La base usa SQLite con Drizzle y habilita WAL y foreign keys. Las migrations se ejecutan como parte de la inicialización del servicio de database.

El schema de sesión contiene tablas normalizadas para el modelo existente y estructuras nuevas, incluyendo conceptos equivalentes a:

- sesión;
- mensajes;
- parts;
- todos;
- `session_message`;
- `session_input`;
- `session_context_epoch`.

**[CONFIRMADO]** Las relaciones usan IDs estables y foreign keys; el modelo de sesión almacena también información como parentage, directorio, título, agente/modelo, versión, resumen/share, coste/tokens, revert/permissions y timestamps según la generación de tabla/campo.

## Projectors

`packages/core/src/session/projector.ts` implementa `SessionProjector`.

**[CONFIRMADO]** El projector consume eventos de sesión V1 como:

- created/updated/deleted;
- message updated/removed;
- part updated/removed.

También procesa eventos más nuevos relacionados con:

- agent/model switching;
- prompts admitidos/ejecutados;
- context updates;
- mensajes sintéticos;
- shell lifecycle;
- step started/ended/failed.

Sus handlers materializan/actualizan proyecciones SQLite.

**[CONFIRMADO]** En el modelo nuevo, la secuencia durable del evento se utiliza para ordenar determinadas proyecciones (`session_message.seq`). El projector también deriva usage/cost desde información de finalización de steps.

## `EventV2`

`packages/core/src/event.ts` define el runtime de eventos V2.

**[CONFIRMADO]** Los eventos durables tienen noción de aggregate y secuencia. El servicio ofrece mecanismos para publicación/commit y proyección dentro de operaciones coordinadas con la database.

**[INFERENCIA]** El objetivo arquitectónico es aproximar la sesión a un modelo event-driven donde el log durable preserva orden causal y las tablas de consulta son proyecciones derivadas, aunque `dev` todavía mantiene writes y contratos heredados.

## Bridge V1 → V2

`packages/opencode/src/event-v2-bridge.ts` conecta eventos del runtime legacy con el nuevo event system.

**[CONFIRMADO]** La existencia y provisión activa del bridge significa que cambios originados en APIs V1 pueden entrar en el pipeline V2 sin obligar a reescribir simultáneamente todos los productores.

**[INFERENCIA]** Este bridge es una pieza de strangler migration: desacopla la sustitución del write model de la sustitución de clientes y producers.

## Storage legacy

`packages/opencode/src/storage/storage.ts` continúa existiendo y es provisto por el server junto con `Database.Service`.

**[CONFIRMADO]** Por tanto `dev` todavía tiene más de una abstracción de persistencia activa.

**[INFERENCIA]** SQLite es el destino de convergencia para estado estructurado, mientras `Storage` conserva compatibilidad con datos/configuraciones o dominios aún no migrados. No debe asumirse que todo el storage legacy haya sido eliminado.

## Lifecycle de datos durante un turn

Flujo reconstruido:

```text
input del usuario
    |
    v
Session / message-input write
    |
    +--> evento(s)
    |
    v
SessionPrompt / SessionProcessor
    |
    +--> message parts / tool states / step states
    |        |
    |        +--> eventos legacy y/o V2
    |                    |
    |                    v
    |              SessionProjector
    |                    |
    v                    v
continuación          SQLite views
    |                    |
    +-----------> API/event stream
                         |
                         v
                    clientes UI
```

## Status y serialización

`session/status.ts` y `session/run-state.ts` separan el estado observable de una sesión de la coordinación interna de su ejecución.

**[CONFIRMADO]** Existe state/lifecycle explícito para impedir que el loop sea simplemente una función stateless. Abort, retry, compaction y tool calls deben coordinarse con el estado de ejecución de la sesión.

## Revert, compaction y summary

Los módulos `revert.ts`, `compaction.ts` y `summary.ts` modifican cómo se reconstruye la historia visible/consumible.

**[CONFIRMADO]** Un revert no equivale necesariamente a borrar físicamente todos los datos históricos; el runtime conserva metadatos que permiten determinar qué porción del historial es efectiva. De forma análoga, compaction produce una representación resumida/sintética para controlar el contexto del modelo.

## Recovery y replay

**[CONFIRMADO]** Al ser mensajes/parts/session state persistentes, los clientes pueden volver a consultar una sesión y el runtime puede reconstruir un contexto después de reinicios. La TUI dispone además de lógica explícita de replay visual al reanudar.

**[PENDIENTE]** Para afirmar garantías exactas de crash consistency entre cada evento, cada part y cada proyección sería necesario ejecutar pruebas de fallo en puntos concretos de commit. El código demuestra transacciones en varias rutas V2, pero no conviene extrapolar atomicidad global a todo el pipeline legacy.

## Invariantes útiles para reverse engineering

1. **[CONFIRMADO]** Message y Part son entidades persistentes, no únicamente chunks de UI.
2. **[CONFIRMADO]** Una sesión puede tener parent, habilitando árboles de subagentes/forks.
3. **[CONFIRMADO]** Event ordering es explícito en la generación V2.
4. **[CONFIRMADO]** Las proyecciones permiten desacoplar eventos de read models.
5. **[INFERENCIA]** El modelo final apunta a una sesión event-sourced o, como mínimo, event-centric con materialized projections, pero `dev` aún no permite denominar a todo el sistema “event sourced” sin matices.