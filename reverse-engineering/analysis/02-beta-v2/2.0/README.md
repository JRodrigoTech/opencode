# Branch `2.0`: exploración del modelo de sesión

## Clasificación

`2.0` no fue una reescritura integral de OpenCode. La evidencia disponible la sitúa como una exploración concreta del modelo de sesión que posteriormente quedó absorbida/reemplazada por diseños más elaborados.

## Hechos demostrables

### Identidad y propósito

- HEAD analizado: `7a6ce05d0939826aa6c8e1c481489a713b2d633f`.
- Commit: `2.0 exploration (#22335)`.
- Fecha: 2026-04-13.
- El PR upstream #22335, `2.0 exploration`, fue merged a `dev`.
- Frente a `beta/v2`, esta branch queda miles de commits por detrás; no representa la V2 completa de agosto de 2026.

### Alcance real del cambio

El PR #22335 modifica sólo seis paths:

```text
packages/opencode/src/id/id.ts
packages/opencode/src/session/projectors.ts
packages/opencode/src/session/session.sql.ts
packages/opencode/src/v2/message.ts
packages/opencode/src/v2/session-entry.ts
packages/opencode/src/v2/session.ts
```

Por tanto, el área experimental fue principalmente **session/message persistence semantics**.

## Modelo introducido

### De `Message` a `SessionEntry`

El experimento elimina `packages/opencode/src/v2/message.ts` e introduce:

```text
packages/opencode/src/v2/session-entry.ts
```

Se crea además un nuevo prefijo de ID:

```text
entry -> ent
```

`SessionEntry` modela una secuencia heterogénea de elementos asociados a una sesión.

Entre las clases observadas están:

- `User`
- `Synthetic`
- `Request`
- `Text`
- `Reasoning`
- `Tool`
- `Complete`

### Tool lifecycle integrado

`SessionEntry.Tool` incorpora estados explícitos:

```text
pending
running
completed
error
```

Los estados contienen elementos que posteriormente aparecen como preocupaciones estables del runtime:

- input estructurado;
- raw/provider input durante parsing;
- output;
- metadata;
- attachments;
- timestamps;
- error terminal;
- pruning metadata.

Esto muestra un intento temprano de convertir actividad del assistant/tool en registros con identidad durable, en vez de depender sólo del stream temporal del provider.

## Persistencia propuesta

`session.sql.ts` contiene una tabla experimental `SessionEntryTable`, aunque comentada:

```text
session_entry
  id
  session_id
  type
  timestamps
  data JSON
```

También se añade un projector experimental desde `MessageV2.Event.PartUpdated` hacia `SessionEntry`, igualmente comentado.

### Lectura

**HECHO.** La persistencia de `SessionEntry` no estaba terminada en el propio PR: tanto tabla como projector aparecen como código experimental/comentado.

**INFERENCIA fuerte.** El branch estaba probando si messages, reasoning, tool calls y terminación podían representarse mediante una única secuencia durable de entries antes de comprometer el schema de base de datos.

## Relación con `dev`

El PR #22335 fue merged, pero eso no significa que su forma final sobreviviera.

### Qué no sobrevivió

**HECHO.** En `dev` actual no existe:

```text
packages/opencode/src/v2/session-entry.ts
```

**HECHO.** Tampoco existe ese path en `beta/v2` actual.

Por tanto, la implementación concreta `SessionEntry` fue posteriormente eliminada o sustituida.

### Qué sí parece haber sobrevivido conceptualmente

V2 posterior formaliza varias de las mismas preocupaciones, pero con boundaries diferentes:

- identidad durable de messages/tool calls;
- tool lifecycle explícito;
- reasoning como dato reconocible;
- durable events;
- transactional projections;
- terminal outcomes;
- replay/recovery.

La V2 madura ya no comprime todo bajo una única abstracción `SessionEntry`: separa Session, inbox, messages, durable events, projections y execution state.

## Comparación con el contrato V2 posterior

### `2.0` experimental

```text
Provider/message activity
        |
        v
  SessionEntry union
        |
        v
experimental session_entry table
```

### `beta/v2` posterior

```text
Prompt admission
   |
   v
Durable inbox
   |
   v
Durable session events ---> projectors ---> messages / tool state / claims / instruction state
   |
   +--> replay/recovery
   +--> execution coordinator
```

La diferencia fundamental es que la V2 posterior hace explícitas **admission**, **execution**, **event history** y **projections** como conceptos separados.

## Agent runtime

No hay evidencia en #22335 de una reescritura completa de:

- agent selection;
- subagents;
- prompt orchestration;
- provider routing;
- tool registry;
- retries;
- compaction;
- backend transport.

Por tanto, atribuir esas áreas a `2.0` sería incorrecto.

El cambio afecta indirectamente al runtime al proponer una representación diferente de sus outputs y tool state, pero no sustituye el runtime completo.

## AI stack y providers

No se observa una nueva abstracción de providers ni un nuevo protocolo LLM dentro del alcance de #22335.

La estructura `Request` sí transporta:

```text
model.id
model.providerID
model.variant?
```

pero eso es metadata de ejecución/session entry, no una reescritura del provider stack.

## Backend, SDK y UI

No hay evidencia de cambios relevantes de backend, SDK o UI en el PR que define esta exploración.

**Conclusión:** `2.0` debe clasificarse como un experimento de **session/domain model**, no como una generación completa del producto.

## Persistencia y protocolos

La principal señal arquitectónica es la intención de hacer proyectable la actividad de sesión. Sin embargo:

- la tabla experimental está comentada;
- el projector está comentado;
- el modelo posterior tomó otra forma.

Esto sitúa `2.0` como evidencia arqueológica valiosa de una decisión en evaluación, no como arquitectura consolidada.

## Ideas descartadas o reemplazadas

### Reemplazada: una única unión `SessionEntry` como representación central

La forma concreta desaparece posteriormente.

### Conservada: actividad del agente como datos con lifecycle e identidad

La V2 posterior profundiza exactamente esa propiedad mediante durable events y projections.

### Reemplazada: projector directo desde `MessageV2.PartUpdated`

La arquitectura posterior convierte los eventos durables de sesión en el boundary semántico central y distingue live deltas de durable records.

## Relación evolutiva

```text
dev temprano
   |
   +--> 2.0 exploration (#22335)
   |      - SessionEntry
   |      - tool state dentro del entry
   |      - intento de projection/table
   |
   +--> múltiples refactors posteriores
          |
          v
       V2 madura
       - durable inbox
       - durable events
       - projections
       - messages
       - execution claims
       - replay/recovery
```

## Hipótesis

1. **INFERENCIA:** `2.0` funcionó como spike para descubrir qué información debía ser durable en una sesión antes de diseñar el modelo V2 definitivo.
2. **INFERENCIA:** la eliminación de `SessionEntry` no representa abandono del enfoque event-oriented; representa lo contrario: las responsabilidades se separaron en primitives más específicas.
3. **INFERENCIA:** la unión inicial era demasiado amplia para actuar simultáneamente como transcript, execution log, tool state y persistence boundary, motivo plausible para la posterior separación. No se encontró una declaración del autor que confirme explícitamente ese motivo.

## Conclusión

`2.0` es importante no porque sea una versión perdida completa de OpenCode, sino porque captura una fase de diseño donde el equipo estaba intentando redefinir **qué es una sesión** y cómo representar de forma durable la actividad del modelo y las herramientas. Su implementación específica fue reemplazada, pero muchas de sus preocupaciones reaparecen de forma más rigurosa en `beta/v2`.
