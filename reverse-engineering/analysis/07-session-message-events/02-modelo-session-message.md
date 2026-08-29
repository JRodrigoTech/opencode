# Modelo de Session y Message

## 1. Dos generaciones conviven en `dev`

OpenCode mantiene actualmente una capa Session de compatibilidad histórica (`SessionV1`) y una línea V2 en `packages/core/src/session*`. No es correcto describir el sistema como si existiera un único modelo homogéneo.

**Hecho confirmado:** `packages/opencode/src/session/session.ts` importa simultáneamente `SessionV1`, `SessionV2`, `EventV2Bridge`, tablas SQLite compartidas y tipos V2 de Project/Workspace/Model.

**Inferencia:** `dev` está en una fase de migración incremental donde la API/runtime legado sigue siendo una superficie funcional mientras la infraestructura durable V2 asume ordering, admission, sync y nuevos boundaries.

---

## 2. Modelo de Session

La estructura `Info` de `packages/opencode/src/session/session.ts` contiene, entre otros:

- `id`
- `slug`
- `projectID`
- `workspaceID?`
- `directory`
- `path?`
- `parentID?`
- `title`
- `agent?`
- `model?`
- `version`
- `metadata?`
- `permission?`
- `summary?`
- `cost?`
- `tokens?`
- `share?`
- `revert?`
- timestamps `created`, `updated`, `compacting?`, `archived?`

La conversión `fromRow()` / `toRow()` muestra que la Session no es sólo una conversación textual: es también una entidad de ownership, ubicación, ejecución y accounting.

### 2.1 Metadata

`Metadata` es `Record<string, any>` en la superficie legacy. Se persiste en `SessionTable.metadata` y puede reemplazarse mediante `setMetadata()`.

**Hecho confirmado:** la metadata de Session es extensible y durable.

**Inferencia:** se usa como escape hatch de compatibilidad/extensión, mientras campos con invariantes fuertes —Project, Workspace, agent, model, permission, cost, tokens— tienen columnas/tipos propios.

### 2.2 Project, Workspace y location

Una Session está ligada a:

- un `projectID`;
- opcionalmente un `workspaceID`;
- un `directory` absoluto;
- opcionalmente un `path` relativo al worktree.

El código calcula `path` con `sessionPath(worktree, cwd)` y la línea V2 añade eventos de movimiento/location.

**Hecho confirmado:** la ubicación forma parte de la identidad operativa de la Session, pero no es su identidad primaria: la misma Session puede sobrevivir a cambios de location y puede ser recuperada incluso si un worktree desaparece.

---

## 3. Parent/child Sessions

`parentID` es un campo opcional de Session.

`children(parentID)` consulta `SessionTable.parent_id`. `remove(sessionID)` obtiene sus children y los elimina recursivamente antes de retirar la Session padre.

La creación usa títulos diferentes:

- root: `New session - <timestamp>`;
- child: `Child session - <timestamp>`.

### Invariante

```text
Session(parentID = X)
        |
        +--> relación estructural con Session X
```

**Hecho confirmado:** parent/child es una relación explícita entre entidades Session y participa en lifecycle de borrado.

**Inferencia:** esta relación está orientada a delegación/subagents/child work, no a versionado de historia. El mecanismo de fork usa una semántica distinta.

---

## 4. Fork no es child Session

`Session.fork()` en `packages/opencode/src/session/session.ts`:

1. lee la Session original;
2. crea una nueva Session con título `(... fork #N)`;
3. conserva `workspaceID` y clona `metadata`;
4. obtiene los mensajes del origen;
5. opcionalmente corta la copia en un `messageID` dado;
6. genera **nuevos Message IDs**;
7. mantiene un `idMap` old → new;
8. reescribe `assistant.parentID` al ID clonado correspondiente;
9. genera **nuevos Part IDs**;
10. reescribe `messageID`/`sessionID` de cada Part;
11. reescribe referencias especiales como `compaction.tail_start_id` mediante el mismo mapping.

No asigna `parentID` de Session a la Session original.

### Resultado

```text
Session A                    Session B (fork)
---------                    ----------------
msg_a1  ------------------>  msg_b1
msg_a2(parent=a1) -------->  msg_b2(parent=b1)
part_a ------------------->  part_b

A y B poseen timelines independientes.
```

**Hecho confirmado:** fork es una copia semántica con identidad nueva, no un alias ni una vista sobre el timeline original.

**Inferencia:** remapear IDs elimina dependencia futura del fork respecto a mutations/deletes del origen y permite que ambos timelines evolucionen independientemente.

---

## 5. Modelo de Message legacy/V1

`packages/opencode/src/session/message-v2.ts` trabaja sobre tipos de `@opencode-ai/core/v1/session` y almacena Message + Part en `MessageTable` / `PartTable`.

Las dos categorías principales son:

- `User`
- `Assistant`

Un assistant message incorpora información suficiente para reconstruir su ejecución: parent, provider/model, agent, timestamps, usage/cost, finish/error y sus Parts.

### Message + Parts

La unidad de lectura habitual es `WithParts`:

```text
Message.info
  + Parts[]
```

`hydrate()` carga las filas `PartTable` de los messages seleccionados y las agrupa por `message_id`.

Los Parts modelan contenido y lifecycle más fino que el Message. Entre los tipos observables en esta generación están:

- text;
- reasoning;
- tool;
- file;
- step-start / step-finish;
- compaction;
- subtask;
- snapshot/patch y otros tipos de control/historia.

**Hecho confirmado:** el Message no es un blob de texto. Es un aggregate de Parts heterogéneos que preservan estructura de provider, tools, reasoning y checkpoints.

---

## 6. Tool state dentro del mensaje

La serialización hacia model messages revela al menos estos estados persistibles del Part `tool`:

```text
pending -> running -> completed
                   \-> error
```

Al reconstruir contexto para el modelo:

- `completed` produce tool output;
- `error` produce output-error o un output parcial si la interrupción preservó salida;
- `pending`/`running` antiguos se convierten en un error sintético de interrupción para evitar un `tool_use` sin `tool_result` al reanudar contexto.

**Hecho confirmado:** la persistencia soporta recuperación de conversaciones que quedaron interrumpidas durante un tool call.

**Inferencia:** el modelo persistido está diseñado para poder transformarse en un transcript válido para providers incluso después de crash/cancel, no sólo para reproducir la UI.

---

## 7. Paginación y orden del modelo legacy

`MessageV2.page()` usa un cursor compuesto por:

- `time`
- `id`

La comparación de filas antiguas es:

```text
time_created < cursor.time
OR
(time_created == cursor.time AND id < cursor.id)
```

Esto proporciona un tie-break determinista para la tabla legacy.

Sin embargo, la proyección V2 `session_message` usa `seq` durable por Session como orden autoritativo.

**Conclusión:** existen dos mecanismos de orden según la generación del read model:

- V1: timestamp + ID;
- V2: aggregate sequence.

No deben confundirse.

---

## 8. Separación Message ID / Event ID

La branch `refactor/core-v2-message-identity` —commit `f2d133edf7430f3229a84c29d6b0e1690f1f2a60`— introdujo:

```text
Event ID   = evt_...
Message ID = msg_...
```

con funciones deterministas:

```text
SessionMessage.ID.fromEvent(evt)
SessionMessage.ID.toEvent(msg)
```

Antes, parte del V2 trataba el Event ID como Message ID.

**Hecho confirmado:** la identidad fue separada deliberadamente y obligó a cambiar inbox, projectors y runner.

**Inferencia:** un evento es una ocurrencia en el log; un mensaje es una entidad de dominio proyectada. Aunque exista correspondencia 1:1 para determinados eventos de creación, sus namespaces no deben acoplarse.

---

## 9. Codec de fila persistida

La branch `message-row-codec`, commit `ffaaad89a44fd4069c8f5f232c9cfaea1163dc6b`, añade `SessionMessageRow` para centralizar:

```text
Domain Message <-> { id, type, data } SQLite row
```

El test de la branch verifica que `id` y `type` de las columnas canónicas prevalecen sobre valores stale dentro de `data`.

**Hecho confirmado:** la identidad/discriminante físicos se consideran autoridad sobre duplicados embebidos en JSON.

**Inferencia:** esta normalización reduce riesgos durante migraciones y schema evolution: el JSON deja de actuar como segunda fuente de verdad para claves estructurales.

---

## 10. Session metadata vs execution state

No toda información asociada a una Session es durable.

### Durable

Ejemplos:

- title;
- metadata;
- agent/model seleccionado;
- project/workspace/location;
- permission;
- usage/cost;
- archive timestamp;
- revert metadata;
- Message/Parts;
- durable Session Events V2.

### Efímero

Ejemplos:

- `busy/retry/idle` de `run-state.ts`/`status.ts`;
- draft del prompt en la rama `session-prompt-drafts`;
- fragmentos live-only de stream antes de su `Ended` durable.

**Conclusión arquitectónica:** OpenCode separa progresivamente tres clases de estado:

1. **domain state durable**;
2. **execution state transitorio**;
3. **presentation/input state local del cliente**.

Esta separación es clave para interpretar correctamente recovery y sincronización.