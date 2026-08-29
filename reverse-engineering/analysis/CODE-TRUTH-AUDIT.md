# Auditoría de verdad documental contra `dev`

Fecha de auditoría: 2026-08-29  
Baseline de código: `dev@dc4449df0d52199704ea4989a5a993ebbc605612`

## Propósito

Este documento registra correcciones transversales detectadas al contrastar `reverse-engineering/analysis` con el source real de `dev`.

La prioridad epistemológica usada es:

1. composition root y wiring de `dev`;
2. implementación/schemas/tests de `dev`;
3. commits/branches históricas;
4. inferencias arquitectónicas explícitamente marcadas.

El nombre de una branch o la mera presencia de código en `dev` **no prueban que ese código sea el path por defecto del producto**.

## Hallazgo transversal 1 — `dev` es híbrido

La principal fuente de errores documentales era describir migraciones parcialmente integradas como sustituciones completas.

### Session/prompt

Coexisten:

- `packages/opencode/src/session/prompt.ts` + `processor.ts` + `llm.ts`;
- `packages/core/src/session/runner/*`.

El primero sigue cableado por `packages/opencode` y no puede etiquetarse como puramente histórico. El segundo es una implementación V2 real, pero sus propiedades no deben proyectarse automáticamente al primero.

### Context/instructions

Coexisten:

- `SystemPrompt` + `Instruction.Service` en `packages/opencode`;
- `SystemContext` + `InstructionContext` + `SessionContextEpoch` en Core V2.

### Compaction/overflow

Coexisten:

- `packages/opencode/src/session/compaction.ts` + `overflow.ts`;
- `packages/core/src/session/compaction.ts` + recovery del runner V2.

Sus budgets, triggers y retries no son idénticos.

### LLM

Coexisten:

- AI SDK como executor por defecto de `packages/opencode/src/session/llm.ts`;
- native `LLMNativeRuntime` opt-in mediante `experimentalNativeLlm` con fallback;
- Core V2 usando `LLMClient`/`@opencode-ai/llm` directamente.

Por tanto, `dev` no ha eliminado globalmente AI SDK.

### Server

Coexisten:

- host/listener/composition root en `packages/opencode/src/server`;
- APIs/handlers extraídos a `packages/server`;
- contracts extraídos a `packages/protocol`;
- services legacy-compatible y Core V2 dentro del mismo graph.

`packages/opencode/src/server/server.ts` vigente usa Effect + Node; Hono corresponde a generaciones históricas.

## Hallazgo transversal 2 — ordering depende del dominio

No existe un único orden por timestamp o ID.

- durable EventV2: `aggregateID + seq`;
- aggregate nuevo: `latest=-1`, primer `seq=0`;
- SessionMessage V2: `seq` derivado del durable event;
- SessionV2 list: `time_created` + ID tie-breaker;
- surfaces legacy pueden utilizar otras claves;
- live chunks se ordenan en el stream pero no son necesariamente durable.

## Hallazgo transversal 3 — published/live no equivale a durable

El stream global, los deltas y el event history cumplen funciones distintas.

- global SSE: seguimiento live/publicado;
- Session history/events: aggregate durable/replayable;
- `EventV2Bridge`: adapta EventV2 a `GlobalBus` y añade payload `sync` para facts durable.

Un reconnect no puede asumir que cada delta live exista como entrada durable individual.

## Hallazgo transversal 4 — schema físico debe comprobarse antes de documentar

Correcciones concretas:

- `event` no tiene timestamp de creación en `dev`; contiene `id`, `aggregate_id`, `seq`, `type`, `data`;
- `event_sequence` contiene `aggregate_id`, `seq`, `owner_id`;
- `SessionStore` vigente no posee `list()` ni `messages()` paginados; esas queries viven hoy en `SessionV2.Service`;
- el host actual expone la especificación OpenAPI en `GET /doc`; no se encontró `/openapi.json` en `dev`.

## Hallazgo transversal 5 — replay durable es estricto

EventV2 sólo acepta un replay sobre sequence ya existente cuando coinciden exactamente:

- ID;
- type persistido/versionado;
- data.

También valida sequence, aggregate y ownership según el modo. No debe describirse como deduplicación tolerante de payloads “equivalentes”.

## Áreas corregidas durante esta auditoría

### `04-prompt-context-compaction`

Corregidos:

- README general;
- SystemContext/context epoch;
- instructions;
- compaction;
- overflow/retry;
- prompt pipeline/continuity.

Cambio principal: separación explícita del pipeline `packages/opencode` y el pipeline Core V2.

### `06-providers-llm-responses`

Corregidos:

- README general;
- migración native/remove-ai-sdk.

Cambio principal: AI SDK sigue siendo runtime por defecto de `SessionPrompt`; native path es opt-in/fallback-aware mientras Core V2 ya consume el stack nativo.

### `07-session-message-events`

Corregidos:

- README general;
- eventos/streaming/ordering;
- persistencia/SQLite;
- sincronización de clientes.

Cambios principales: `seq` inicial, schema físico de event tables, ownership real de SessionStore y ordering real de SessionV2.

### `09-backend-transports`

Corregidos:

- README general;
- arquitectura del servidor;
- HTTP API/endpoints.

Cambio principal: el host actual sigue en `packages/opencode`; `packages/server`/`packages/protocol` son extracción real pero parcial. OpenAPI actual: `/doc`.

## Áreas inspeccionadas sin corrección material en esta pasada

### `03-agent-subagent-skills`

Se verificó contra:

- `packages/opencode/src/agent/agent.ts`;
- `packages/opencode/src/tool/task.ts`;
- `packages/opencode/src/agent/subagent-permissions.ts`.

Las afirmaciones centrales sobre child Sessions, `task_id`, model fallback y herencia de deny/external_directory coinciden con `dev`.

### `08-mcp-acp`

Se verificó específicamente la afirmación de que `unstable_createElicitation` no está en `dev`; la búsqueda actual no devuelve implementación.

### `01-dev`, `02-beta-v2`, `05-tools-permissions-codemode`, `10-effect-modularization`

La pasada transversal no detectó una contradicción arquitectónica del mismo nivel que las corregidas arriba. Sus afirmaciones sobre coexistencia/migración son, en general, compatibles con el wiring observado. Esto no convierte cada frase histórica en una garantía eterna: deben seguir auditándose cuando cambie el baseline `dev`.

## Reglas para futuros agentes

1. **Antes de decir “actual”, localizar el composition root que consume el componente.**
2. **“Existe en `dev`” y “es el default runtime” son afirmaciones distintas.**
3. **No convertir una branch de migración en estado vigente sin comprobar su integración.**
4. **No mezclar V1/legacy-compatible y V2/Core sin etiquetar el pipeline.**
5. **No inferir ordering desde IDs cuando existe sequence/cursor explícito.**
6. **Comprobar schemas físicos antes de enumerar columnas.**
7. **Distinguir live/published/durable/replayable.**
8. **Distinguir protocol contract, server implementation y application host.**
9. **Cuando un dato procede de una branch, marcarlo HISTÓRICO aunque la idea sobreviva en `dev`.**
10. **Una inferencia arquitectónica nunca debe redactarse como HECHO si el source sólo la sugiere.**

## Estado de la auditoría

La documentación corregida queda alineada con el baseline indicado. Si `dev` avanza, este documento debe actualizar el SHA y volver a validar especialmente composition roots, flags experimentales, schemas y mappings de migración, porque son las zonas con mayor probabilidad de cambiar la interpretación arquitectónica.