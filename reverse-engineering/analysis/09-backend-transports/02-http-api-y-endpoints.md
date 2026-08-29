# 02 — HTTP API y endpoints

## 1. Fuente de verdad en `dev`

**Hecho confirmado.** La superficie HTTP vigente se declara en `packages/protocol/src/api.ts` y `packages/protocol/src/groups/*.ts`. Los handlers concretos viven en `packages/server/src/handlers/*.ts`.

Esto cambia radicalmente respecto de la generación Hono: el inventario de endpoints deja de inferirse de llamadas imperativas a `.get()`, `.post()`, etc. y pasa a formar parte del contrato tipado.

## 2. Grupos del API

El `HttpApi` actual compone estos dominios:

| Grupo | Responsabilidad |
| --- | --- |
| `health` | readiness/health del servidor |
| `location` | resolución de directory/workspace |
| `agent` | catálogo de agents |
| `session` | lifecycle y ejecución de sesiones |
| `message` | superficie de mensajes |
| `model` | modelos y selección |
| `provider` | providers |
| `integration` | integraciones |
| `credential` | credenciales |
| `permission` | requests/respuestas de permisos |
| `fs` | filesystem |
| `command` | commands |
| `skill` | skills |
| `event` | stream global de eventos |
| `pty` | PTY CRUD + WebSocket |
| `question` | preguntas/elicitation de UI |
| `reference` | referencias |
| `project-copy` | operaciones de copia/proyecto |

La lista exacta se deriva de `packages/protocol/src/api.ts` en `dev`.

## 3. Endpoints de infraestructura y transport

### Health

`packages/protocol/src/groups/health.ts`:

- `GET /api/health`
  - success: `{ healthy: true }`
  - identificador OpenAPI: `v2.health.get`.

El health endpoint es una dependencia del discovery de daemon/desktop; no es sólo observabilidad.

### Location

`packages/protocol/src/groups/location.ts`:

- `GET /api/location`
  - query `location[directory]`, `location[workspace]` mediante deep-object.

### Agent

`packages/protocol/src/groups/agent.ts`:

- `GET /api/agent`
  - location-scoped.

### Event global

`packages/protocol/src/groups/event.ts`:

- `GET /api/event`
  - success `StreamSse`.

Su semántica se detalla en `03-eventos-websocket-rpc.md`.

### PTY

`packages/protocol/src/groups/pty.ts`:

- `GET /api/pty`
- `POST /api/pty`
- `GET /api/pty/:ptyID`
- `PUT /api/pty/:ptyID`
- `DELETE /api/pty/:ptyID`
- `POST /api/pty/:ptyID/connect-token`
- `GET /api/pty/:ptyID/connect` — upgrade WebSocket.

## 4. Session API: la superficie más significativa para sincronización

`packages/protocol/src/groups/session.ts` define actualmente:

| Método | Path | Semántica |
| --- | --- | --- |
| GET | `/api/session` | listado paginado |
| POST | `/api/session` | crear sesión |
| GET | `/api/session/active` | drains/ejecuciones activas en este proceso |
| GET | `/api/session/:sessionID` | recuperar sesión |
| POST | `/api/session/:sessionID/agent` | cambiar agent para turns posteriores |
| POST | `/api/session/:sessionID/model` | cambiar model |
| POST | `/api/session/:sessionID/prompt` | admitir durablemente input y programar agent loop |
| POST | `/api/session/:sessionID/compact` | compactar contexto |
| POST | `/api/session/:sessionID/wait` | esperar idle |
| POST | `/api/session/:sessionID/revert/stage` | preparar revert |
| POST | `/api/session/:sessionID/revert/clear` | limpiar revert |
| POST | `/api/session/:sessionID/revert/commit` | confirmar revert |
| GET | `/api/session/:sessionID/context` | contexto activo |
| GET | `/api/session/:sessionID/history` | página finita de eventos durables |
| GET | `/api/session/:sessionID/event` | SSE durable con replay por sequence |
| POST | `/api/session/:sessionID/interrupt` | interrumpir ejecución activa |
| GET | `/api/session/:sessionID/message/:messageID` | proyección de un mensaje |

### Paginación

**Hecho confirmado.** El listado usa un cursor opaco codificado Base64URL que contiene anchor y dirección. El servidor reconstruye `previous` y `next` usando ID y timestamp del primer/último elemento de la página.

### Middleware de sesión

Los endpoints de una sesión concreta incorporan un middleware de resolución de location/session. Esto evita que el cliente tenga que repetir toda la ubicación para cada operación sobre un `sessionID` ya identificado.

### Semántica de prompt

El contrato de `session.prompt` no describe una llamada LLM sincrónica. Declara explícitamente admisión durable de input y scheduling del agent loop. Esta separación explica por qué el cliente se sincroniza mediante eventos y proyecciones en lugar de esperar que el POST transporte toda la ejecución.

## 5. OpenAPI como producto del contrato

**Hecho confirmado.** `packages/server/src/routes.ts` monta la especificación OpenAPI en `/openapi.json` a partir del `HttpApi`.

Consecuencias:

- schema y runtime comparten fuente de verdad;
- los adapters/clientes pueden derivarse del contrato;
- la migración histórica Hono→HttpApi dejó de necesitar un inventario manual permanente.

## 6. La migración histórica de rutas

### Generación Hono

En `kit/httpapi-route-inventory-current`, commit `2b028287e21ae5931d6654738872c7b5ba9bc55a`, todavía se inspeccionan las registrations Hono para producir una tabla de paridad.

El script clasificaba rutas como:

- `bridged`: ya servidas por HttpApi mediante bridge;
- `next`: próximas a portar;
- `later`: todavía Hono;
- `special`: SSE, WebSocket o UI bridge.

Entre las superficies visibles entonces estaban `/agent`, `/command`, `/config`, `/event`, `/file`, `/mcp`, `/path`, `/permission`, `/project`, `/provider`, `/pty`, `/session`, `/sync` y rutas experimentales.

### `backend-adapter-v2`

El commit `f346024f0e0d1d98bfa83c8f1216bf988af2abc1` mueve la provisión de `HttpRouter.layer` para compartir router entre API y fallback. Es evidencia concreta de una etapa donde el nuevo HttpApi y las rutas legacy convivían en el mismo proceso.

### Resultado

En `dev`, `packages/protocol`/`packages/server` ya son la estructura primaria y no una isla bridged dentro de Hono.

## 7. Authentication boundary

### Listener auth

`packages/server/src/auth.ts` implementa Basic Auth para el servidor cuando hay password configurado.

- username configurable;
- default: `opencode`;
- password habilita el gate.

### PTY ticket exception

El connect WebSocket de PTY puede atravesar el middleware de credenciales mediante un ticket específico. La responsabilidad de autorización se desplaza entonces al handler, que consume el ticket y valida origin/scope.

**Conclusión:** no se trata de una ruta “sin auth”; usa una credencial de capability más estrecha y de un solo uso.

## 8. Boundary HTTP vs dominio

**Hecho confirmado.** Los handlers transforman errores de core en errores de protocol (`SessionNotFoundError`, `ConflictError`, `ServiceUnavailableError`, etc.) y transforman objetos de dominio en responses tipadas.

**Inferencia.** Esto convierte al HTTP API en una anti-corruption layer deliberada: los clientes no necesitan conocer tags internos de Effect/core ni las representaciones exactas del storage.

## 9. Comparativa con la arquitectura anterior

| Aspecto | Hono histórico | `dev` |
| --- | --- | --- |
| Definición de rutas | imperativa en server modules | declarativa en protocol groups |
| Schemas | ligados a handlers/OpenAPI helpers | parte del endpoint contract |
| OpenAPI | generado desde registrations/metadata | generado desde HttpApi |
| SDK | dependiente de surface histórica | derivable del mismo contrato |
| Errors | más próximos al implementation | protocol errors explícitos |
| Streaming | rutas especiales Hono | `StreamSse`/raw WebSocket tipados |
| Fallback | necesario durante migration | ya no es el centro del diseño |

## 10. Hechos e inferencias

### Confirmado

Los paths y semánticas enumerados para health/location/agent/session/event/PTY proceden directamente de los grupos de `protocol` en `dev`.

### Inferencia

El `HttpApi` está diseñado para ser la interfaz estable entre runtime y múltiples clientes. Aunque no todos los consumers tengan idéntico nivel de code generation, la separación física y los experimentos de adapter/RPC sustentan esta lectura.