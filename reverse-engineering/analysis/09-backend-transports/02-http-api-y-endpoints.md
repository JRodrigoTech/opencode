# 02 — HTTP API y endpoints

## 1. Fuente de verdad en `dev`: superficie compuesta

La superficie HTTP vigente **no procede de un único package**.

El host real está en:

- `packages/opencode/src/server/server.ts`;
- `packages/opencode/src/server/routes/instance/httpapi/server.ts`.

Ese composition root monta simultáneamente:

1. APIs/grupos/handlers definidos todavía bajo `packages/opencode/src/server/routes/instance/httpapi/*`;
2. `Api` + handlers extraídos de `@opencode-ai/server`;
3. contracts reutilizables de `@opencode-ai/protocol` para la surface V2 extraída.

Por tanto, `packages/protocol/src/api.ts` es fuente de verdad de **la API extraída que declara**, pero no debe presentarse como inventario exhaustivo de toda ruta que el host `packages/opencode` sirve actualmente.

## 2. API extraída `@opencode-ai/protocol` / `@opencode-ai/server`

La surface extraída agrupa dominios como health, location, agent, session, message, model, provider, integration, credential, permission, fs, command, skill, event, pty, question, reference y project-copy.

Los handlers correspondientes viven en `packages/server/src/handlers*` y se montan desde:

```text
HttpApiBuilder.layer(Api)
```

dentro del composition root de `packages/opencode`.

## 3. Session V2 API

`packages/server/src/handlers/session.ts` implementa operaciones del grupo Session V2, entre ellas:

- list/create/get;
- active;
- switch agent/model;
- prompt admission;
- compact/wait;
- revert stage/clear/commit;
- context;
- durable history;
- durable event stream;
- interrupt;
- message lookup.

### Semántica de prompt

`session.prompt` admite input durable y activa el lifecycle de ejecución; no representa una llamada LLM síncrona cuya respuesta completa deba viajar dentro del POST.

### History

`session.history` llama a `SessionV2.history()` con `after` y `limit`, y devuelve `{ data, hasMore }`.

### Events

`session.events` devuelve el stream de `SessionV2.events({ sessionID, after })`, respaldado por durable EventV2 del aggregate.

## 4. Event global

`packages/server/src/handlers/event.ts` implementa el stream global SSE de la API extraída:

- subscriber bounded `256`;
- `server.connected` sintético;
- heartbeat cada 15 s;
- payloads codificados por `OpenCodeEvent`.

Este stream es distinto de `session.events`: el primero sigue el bus global publicado; el segundo está scoped al aggregate durable de una Session.

## 5. PTY

PTY mantiene HTTP para lifecycle y un WebSocket específico para conexión interactiva, con auth/ticketing especializado en el composition root.

No debe inferirse de esta excepción que WebSocket sea el transport principal del API.

## 6. Rutas que siguen bajo `packages/opencode`

`packages/opencode/src/server/routes/instance/httpapi/server.ts` monta handlers locales para, entre otros:

- config;
- control/control-plane/global;
- experimental;
- file;
- instance;
- MCP;
- project/project-copy;
- PTY;
- question;
- permission;
- provider;
- session;
- sync;
- TUI;
- workspace.

Algunos dominios se solapan conceptualmente con la API extraída porque representan generaciones/surfaces distintas durante la migración. La existencia de un handler extraído no autoriza a asumir que el homónimo local ya no participa en el backend.

## 7. OpenAPI / documentación

**Corrección:** el host vigente en `packages/opencode/src/server/routes/instance/httpapi/server.ts` crea la especificación mediante:

```text
OpenApi.fromApi(PublicApi)
```

y expone una ruta raw:

```text
GET /doc
```

con la respuesta JSON cacheada.

No se ha encontrado `/openapi.json` en el baseline `dev`; por tanto no debe documentarse como path vigente.

`packages/opencode/src/server/server.ts` también exporta `openapi()` que devuelve `OpenApi.fromApi(PublicApi)` programáticamente.

## 8. Auth boundary

El host aplica authorization layers diferenciadas para:

- API HTTP normal;
- server routes extraídas;
- PTY connect;
- fallback/UI router.

La configuración de auth se centraliza alrededor de `ServerAuth.Config.layer`. La surface exacta no debe reducirse únicamente a un fichero histórico de Basic Auth en `packages/server`; hay middleware vigente en el host `packages/opencode`.

## 9. Location/workspace/session scoping

El composition root aplica capas como:

- workspace routing;
- instance context;
- session/location middleware en packages extraídos;
- authorization/schema-error layers.

Esto confirma un boundary de routing contextual: los handlers de dominio reciben un contexto ya resuelto en vez de reinterpretar directory/workspace en cada operación.

## 10. Evolución Hono → Effect HttpApi

La familia `kit/httpapi-*` documenta una migración incremental desde registrations Hono hacia grupos HttpApi, con bridge/fallback durante etapas intermedias.

El resultado en `dev` no es “todo movido a packages/server”, sino:

```text
Effect HttpApi host en packages/opencode
        +
APIs locales aún vigentes
        +
Api/handlers extraídos de packages/server/protocol
```

## 11. Boundary HTTP vs dominio

Los handlers extraídos transforman errores Core a errores de protocolo tipados (`SessionNotFoundError`, `ConflictError`, `ServiceUnavailableError`, etc.). Los handlers locales realizan adaptaciones equivalentes dentro de su propia surface.

**Inferencia:** la dirección arquitectónica es estabilizar un contrato reusable, pero el anti-corruption layer está repartido durante la migración.

## Invariantes auditados

1. `packages/opencode` sigue siendo el host/composition root del HTTP API.
2. `packages/protocol`/`packages/server` son una extracción real pero parcial.
3. Session V2 posee history + durable event stream explícitos.
4. global SSE y Session durable stream no son el mismo canal.
5. la documentación OpenAPI vigente del host está en `GET /doc`, no `/openapi.json`.
6. Hono pertenece a la historia, no al listener `dev` actual.

## Referencias

- `packages/opencode/src/server/server.ts`
- `packages/opencode/src/server/routes/instance/httpapi/server.ts`
- `packages/server/src/handlers/session.ts`
- `packages/server/src/handlers/event.ts`
- `packages/server/src/api.ts`
- `packages/protocol/src/api.ts`
