# 09 — Backend, transports y procesos distribuidos

## Alcance

Este análisis reconstruye la evolución del backend de OpenCode y sus transports contra `dev@dc4449df0d52199704ea4989a5a993ebbc605612`.

## Corrección de auditoría: el servidor vigente es híbrido

La documentación anterior simplificaba demasiado la migración diciendo que el contrato vive en `packages/protocol` y la implementación vigente en `packages/server`, como si `packages/opencode/src/server` fuese ya sólo historia. Eso no coincide con el composition root actual.

En `dev`:

- `packages/opencode/src/server/server.ts` sigue siendo el **listener/application host vigente**;
- `packages/opencode/src/server/routes/instance/httpapi/server.ts` compone el grafo de rutas y servicios;
- ese composition root monta simultáneamente:
  - rutas/handlers todavía definidos en `packages/opencode`;
  - `Api` y handlers extraídos desde `@opencode-ai/server`;
  - contratos/schemas repartidos entre surfaces locales y packages extraídos;
  - servicios legacy-compatible y V2/Core en el mismo backend.

Por tanto, la arquitectura actual debe describirse como **extracción incremental con composition root en `packages/opencode`**, no como una sustitución consumada por `packages/server`.

## Arquitectura vigente

```text
Node/Effect listener
packages/opencode/src/server/server.ts
              |
              v
HttpApiApp.createRoutes(...)
packages/opencode/src/server/routes/instance/httpapi/server.ts
              |
      +-------+--------------------------+
      |                                  |
      v                                  v
local/root/instance groups        @opencode-ai/server Api
+ handlers in packages/opencode  + extracted handlers
      |                                  |
      +---------------+------------------+
                      v
            application service graph
      SessionPrompt / LLM / MCP / Tools / ...
      EventV2 / SessionProjector / Core services
```

`Server.Default()` expone además el mismo handler mediante `app.fetch` in-process; `Server.listen()` lo monta sobre Node HTTP.

## Listener y lifecycle

`packages/opencode/src/server/server.ts` usa Effect/Node, no Hono en el baseline actual. Gestiona:

- listener HTTP Node;
- selección/fallback de port;
- mDNS;
- scope/lifecycle;
- cierre de conexiones HTTP activas;
- cierre coordinado de WebSockets mediante `WebSocketTracker`;
- handler in-process para clientes embebidos.

La implementación Hono pertenece a generaciones históricas y debe marcarse como tal.

## Contratos y extracción

`packages/protocol` y `packages/server` sí representan boundaries nuevos y reales. `packages/server` implementa grupos extraídos con `HttpApiBuilder`, y `packages/protocol` concentra contratos reutilizables en diversas superficies. Pero la migración todavía se integra desde `packages/opencode`.

**Lectura arquitectónica:** el boundary estable se está desplazando desde un server monolítico hacia contracts + handlers extraídos, mientras el host/composition layer mantiene compatibilidad hasta completar el traslado.

## Eventos y sincronización

Hay al menos dos niveles relevantes:

- stream global de eventos, con una implementación extraída en `packages/server/src/handlers/event.ts` que usa SSE, buffer 256, `server.connected` y heartbeat de 15 s;
- eventos durable/session sync en las surfaces de Session/EventV2, proyectados y puenteados hacia clientes mediante `EventV2Bridge` y rutas de session/history/event según la API usada.

`EventV2Bridge` añade location cuando falta y emite además mensajes `sync` para eventos durable con `seq`, aggregate, type versionado y data.

## PTY y WebSocket

PTY continúa siendo un caso de transporte bidireccional especial con WebSocket/tickets. Esto no convierte WebSocket en el transporte general del backend.

Las branches de WebSocket RPC son evidencia experimental/evolutiva; no sustituyen al HTTP/HttpApi vigente como default.

## Daemon, service y desktop

Las familias `daemon-*`, `service-*` y desktop muestran separación entre:

- proceso supervisor/discovery/auth;
- listener/backend;
- cliente/renderer.

Las garantías concretas de PID, health, auth y restart deben atribuirse al código actual de CLI/desktop o a una branch histórica concreta, no inferirse por nombre.

## Generaciones

| Generación | Idea dominante | Estado frente a `dev` |
|---|---|---|
| Hono monolítico | server/routing/middleware juntos en `packages/opencode` | histórico, sustituido en el listener vigente |
| Migración Effect HttpApi | contract/group/handler extraction | integrada parcialmente y estructura actual |
| `packages/protocol` / `packages/server` | contracts y handlers reutilizables | reales, pero aún montados desde host `packages/opencode` |
| Daemon/service | process ownership/discovery/restart | varias garantías sobreviven en CLI/desktop |
| WebSocket RPC | multiplexar requests/streams | experimental/no default |
| Embedded fetch | atravesar API sin socket físico | vigente como capacidad mediante `Server.Default().app.fetch` |

## Documentos

- [01 — Arquitectura del servidor y boundaries](./01-arquitectura-servidor-y-boundaries.md)
- [02 — HTTP API y endpoints](./02-http-api-y-endpoints.md)
- [03 — Eventos, SSE, WebSocket y RPC](./03-eventos-websocket-rpc.md)
- [04 — Daemon, service lifecycle, discovery y versionado](./04-daemon-service-lifecycle.md)
- [05 — Desktop, embedded server y sincronización](./05-desktop-embedded-synchronization.md)
- [06 — Inventario y clasificación de branches](./06-inventario-branches.md)

## Evidencia primaria

- `packages/opencode/src/server/server.ts`
- `packages/opencode/src/server/routes/instance/httpapi/server.ts`
- `packages/opencode/src/event-v2-bridge.ts`
- `packages/server/src/api.ts`
- `packages/server/src/handlers.ts`
- `packages/server/src/handlers/event.ts`
- `packages/protocol/src/api.ts`
- `packages/core/src/event.ts`

## Conclusión

El backend vigente no es “el antiguo server” ni “ya sólo `@opencode-ai/server`”. Es un **host Effect en `packages/opencode` que integra la nueva arquitectura de contracts/handlers con servicios legacy y V2**. Esa coexistencia es precisamente la evidencia más fuerte de una migración strangler y debe conservarse explícita en la documentación.