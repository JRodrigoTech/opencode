# 09 — Backend, transports y procesos distribuidos

## Alcance

Este análisis reconstruye la evolución de la arquitectura backend de OpenCode y sus transports comparando las familias de branches `server-*`, `service-*`, `daemon-*`, `websocket-*`, `backend-*`, `client-service`, `event-subscribe`, `native-http-*`, `httpapi-*` y ramas próximas de RPC, discovery, sidecars y lifecycle contra `dev`.

Baseline analizado: `dev` en `dc4449df0d52199704ea4989a5a993ebbc605612` (2026-08-29).

La investigación se realizó desde la branch documental `reverse-engineering`; los únicos cambios producidos por este agente están bajo este directorio.

## Conclusión ejecutiva

La evolución relevante no es simplemente “Hono → Effect”. Hay cuatro cambios de arquitectura superpuestos:

1. **Del servidor monolítico al contrato separado.** Las generaciones antiguas concentran rutas, middleware, SSE, WebSocket y OpenAPI en `packages/opencode/src/server/server.ts`, sobre Hono. `dev` separa el contrato en `@opencode-ai/protocol` y la implementación en `@opencode-ai/server`, con `HttpApi` de Effect como fuente tipada de endpoints, errores, middleware y OpenAPI.
2. **Del discovery oportunista a un servicio local autenticado.** `server-discovery` empieza con un `server.json` mínimo (`url`, `pid`). Las familias `daemon-election`, `client-service` y `service-*` introducen ownership, health/version gates, sustitución segura, authenticated stop, restart y watchdogs. `dev` conserva la idea, pero en una implementación `Daemon` más compacta dentro de CLI.
3. **De un único stream global a dos niveles de sincronización.** `dev` ofrece `/api/event` como SSE global y `/api/session/:sessionID/event` como SSE durable por agregado, con history/replay por sequence. El experimento `websocket-rpc` propone multiplexar unary y streaming en un único WebSocket, pero no es la superficie genérica vigente.
4. **Del backend como proceso externo al backend como boundary abstracto.** El desktop puede supervisar un sidecar local, obtener endpoint+credenciales por IPC y conectar el renderer como cliente HTTP. En generaciones embebidas, la misma API se puede invocar in-process mediante un `fetch` sintético, lo que demuestra que el boundary arquitectónico es el contrato HTTP/SDK y no necesariamente un socket físico.

## Arquitectura vigente (`dev`)

```text
                  ┌─────────────────────────────┐
                  │ @opencode-ai/protocol       │
                  │ HttpApi + schemas + errors  │
                  └──────────────┬──────────────┘
                                 │ implemented by
                  ┌──────────────▼──────────────┐
                  │ @opencode-ai/server         │
                  │ handlers + auth + routes    │
                  │ location/session middleware │
                  └───────┬───────────┬─────────┘
                          │ HTTP       │ SSE / WS special
            ┌─────────────▼───┐   ┌───▼────────────────┐
            │ generated/client│   │ event + PTY        │
            │ SDK / adapters   │   │ streaming surfaces │
            └────────┬─────────┘   └────────────────────┘
                     │
       ┌─────────────┼──────────────────────────────┐
       │             │                              │
┌──────▼─────┐ ┌─────▼──────────┐          ┌────────▼────────┐
│ CLI/TUI    │ │ desktop renderer│          │ remote clients  │
│ Daemon     │ │ via Electron IPC│          │ / integrations  │
└──────┬─────┘ └─────┬──────────┘          └─────────────────┘
       │              │
       │        Electron main supervises sidecar
       └──────── service registration / health / auth
```

### Propiedades confirmadas

- El API tipado se compone en `packages/protocol/src/api.ts` a partir de grupos `health`, `location`, `agent`, `session`, `message`, `model`, `provider`, `integration`, `credential`, `permission`, `fs`, `command`, `skill`, `event`, `pty`, `question`, `reference` y `project-copy`.
- `packages/server/src/routes.ts` materializa esos contratos mediante `HttpApiBuilder.layer`, publica OpenAPI en `/openapi.json`, aplica CORS, auth, servicios de dominio y fallback de rutas.
- La autenticación de listener usa Basic Auth cuando existe password; `OPENCODE_SERVER_USERNAME` cae a `opencode` y `OPENCODE_SERVER_PASSWORD` activa la exigencia de credenciales.
- `/api/event` es SSE global: buffer acotado a 256, evento sintético `server.connected`, heartbeat cada 15 s y headers anti-buffering.
- `/api/session/:sessionID/event` expone el stream durable de una sesión; `/api/session/:sessionID/history` permite paginar eventos persistidos por sequence. Esto forma una base explícita para replay/resincronización.
- PTY es una excepción WebSocket deliberada: primero puede emitirse un token corto single-use y el upgrade se realiza en `/api/pty/:ptyID/connect`.
- El daemon actual en `packages/cli/src/services/daemon.ts` persiste un registro local, mantiene password separado con permisos `0600`, comprueba health/auth antes de señalizar procesos y protege contra PID reuse revalidando identidad antes de escalar a `SIGKILL`.
- Electron main no entrega acceso directo al backend interno al renderer: espera `ServerReadyData { url, username, password }` y lo expone mediante IPC; el renderer opera sobre la conexión de cliente.

## Generaciones identificadas

| Generación | Branches representativas | Idea dominante | Resultado frente a `dev` |
| --- | --- | --- | --- |
| Hono monolítico | `refactor/node-server-adapter`, `cleanup-server-routes`, `refactor/server-route-organization` | Rutas y transports dentro de `packages/opencode` | Sustituido por separación protocol/server |
| Migración HttpApi | familia `kit/httpapi-*`, `fix/httpapi-query-schema-drift`, `backend-adapter-v2` | Portar rutas gradualmente, generar inventario/SDK y preservar fallback | Concepto consolidado en `dev` |
| Discovery inicial | `server-discovery` | `server.json` con URL/PID + health | Evoluciona a registro autenticado/versionado |
| Managed service | `daemon-election`, `client-service`, `service-*`, `authenticated-service-stop` | Election, ownership, health/version gate, stop/restart seguro | Parte de las garantías sobrevive en `Daemon` actual |
| Desktop sidecar v2 | `desktop-v2-daemon`, `app-backend-v2`, `backend-adapter-v2` | Backend como servicio supervisado + adapter de aplicación | Principios visibles en desktop actual |
| RPC sobre WebSocket | `websocket-rpc` | Un socket multiplexado para unary + streams | Experimental/aditivo, no default en `dev` |
| Embedded lifecycle | `embedded-server-dispose` | Invocar API in-process y disponer recursos explícitamente | Demuestra boundary lógico independiente del socket |

## Documentos

- [01 — Arquitectura del servidor y boundaries](./01-arquitectura-servidor-y-boundaries.md)
- [02 — HTTP API y endpoints](./02-http-api-y-endpoints.md)
- [03 — Eventos, SSE, WebSocket y RPC](./03-eventos-websocket-rpc.md)
- [04 — Daemon, service lifecycle, discovery y versionado](./04-daemon-service-lifecycle.md)
- [05 — Desktop, embedded server y sincronización](./05-desktop-embedded-synchronization.md)
- [06 — Inventario y clasificación de branches](./06-inventario-branches.md)

## Convención epistemológica

- **Hecho confirmado:** observado directamente en código, contrato, test o commit concreto.
- **Inferencia:** reconstrucción de intención o relación evolutiva no declarada de forma inequívoca por el código.
- **Descartado del subsistema:** branch cuyo nombre coincide con el patrón de búsqueda, pero cuyo cambio específico pertenece a otro transport, por ejemplo WebSocket de proveedor LLM y no WebSocket cliente↔backend.

## Evidencia primaria principal

Baseline `dev`:

- `packages/protocol/src/api.ts`
- `packages/protocol/src/groups/event.ts`
- `packages/protocol/src/groups/session.ts`
- `packages/protocol/src/groups/pty.ts`
- `packages/server/src/api.ts`
- `packages/server/src/routes.ts`
- `packages/server/src/auth.ts`
- `packages/server/src/handlers/event.ts`
- `packages/server/src/handlers/session.ts`
- `packages/server/src/handlers/pty.ts`
- `packages/cli/src/services/daemon.ts`
- `packages/desktop/src/main/index.ts`
- `packages/desktop/src/main/server.ts`
- `packages/desktop/src/preload/types.ts`

Branches históricas / experimentales:

- `refactor/node-server-adapter`: antiguo `packages/opencode/src/server/server.ts`
- `server-discovery`: antiguo `packages/opencode/src/cli/server-discovery.ts`
- `kit/httpapi-route-inventory-current`, commit `2b028287e21ae5931d6654738872c7b5ba9bc55a`
- `app-backend-v2`, commit `712d4fb8003be143a39c6af9d9efe980184da7d1`
- `daemon-election`, commit `c61e93a1f3f6aa5d0f09904c685cd51db0762f4b`
- `service-restart`, commit `eae5eaff5db0e66b92452ce3a3c2456364878e61`
- `service-version-guard`, commit `910af9a122cb828abc4a12555facf82c9f88e149`
- `authenticated-service-stop`, commit `bae7a954a871591211671465f1fc59dac090c72a`
- `service-shutdown-watchdog`, commit `78ed6b212e01b25c7791be54e683d87419c5f2c2`
- `embedded-server-dispose`, commit `31d25c8f084dfe5ee35e855e1e40be4159d0799c`
- `websocket-rpc`, commit `b2763569860285cc555ba401e3d4bf55e2f5702f`
