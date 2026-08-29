# 01 — Arquitectura del servidor y boundaries

## 1. Baseline vigente en `dev`

### Hecho confirmado

El composition root del backend vigente sigue en `packages/opencode`:

- `packages/opencode/src/server/server.ts` posee el listener y expone el handler in-process;
- `packages/opencode/src/server/routes/instance/httpapi/server.ts` compone routes, middleware y service graph.

Al mismo tiempo, parte del contrato/implementación ya está extraída a `packages/protocol` y `packages/server`.

La forma correcta de describir `dev` es por tanto:

```text
packages/opencode server host
        |
        +--> routes/handlers locales todavía no extraídos
        +--> @opencode-ai/server Api + handlers extraídos
        +--> protocol/core contracts y services
```

No es correcto afirmar que el boundary central “ya no” esté en `packages/opencode`: el host real continúa allí.

---

## 2. Listener vigente: Effect + Node

`packages/opencode/src/server/server.ts` usa:

- `@effect/platform-node`;
- `effect/unstable/http`;
- `effect/unstable/httpapi` para OpenAPI;
- `node:http`.

`Server.Default()` obtiene el `webHandler` de `HttpApiApp` y expone:

```text
app.fetch(Request)
app.request(url/request, init)
```

`Server.listen()` monta el mismo application handler sobre un listener Node y devuelve hostname, port, URL y `stop()`.

### Lifecycle confirmado

- si el port solicitado es `0`, primero intenta 4096 y después un port libre;
- cada listener obtiene su propio `ConfigProvider.fromEnv()`;
- el listener posee un `Scope` Effect;
- puede publicar mDNS si procede;
- `stop(true)` fuerza cierre de conexiones HTTP activas;
- `WebSocketTracker` participa en el cierre de WebSockets;
- la URL global se limpia cuando se cierra el scope correspondiente.

Éste es código **vigente en `dev`**, no arquitectura histórica.

---

## 3. Composition root de rutas

`packages/opencode/src/server/routes/instance/httpapi/server.ts` muestra la coexistencia de varias capas.

### Rutas todavía compuestas localmente

Define/monta, entre otras:

- `RootHttpApi`;
- `InstanceHttpApi`;
- `EventApi`;
- `PtyConnectApi`;
- handlers de config, control, experimental, file, global, instance, MCP, permission, project, provider, PTY, question, session, sync, TUI y workspace.

### Rutas extraídas

También monta:

```text
const serverRoutes = HttpApiBuilder.layer(Api)
```

donde `Api` y `handlers` provienen de `@opencode-ai/server`.

Por tanto la extracción no es hipotética, pero tampoco total.

### Service graph

El mismo archivo compone nodos como:

- `SessionPrompt`;
- `SessionProcessor`;
- `SessionCompaction`;
- `LLM`;
- `MCP`;
- `ToolRegistry`;
- `EventV2Bridge`;
- `SessionProjector`;
- `EventV2`;
- Project/Core services y otros.

Esto hace del server host un anti-corruption/composition layer durante la transición.

---

## 4. `packages/protocol` y `packages/server`

### Hecho confirmado

Son packages reales con responsabilidades extraídas:

- `packages/protocol`: contratos/schemas reutilizables;
- `packages/server`: API/handlers/middleware reutilizables para grupos ya migrados.

Estos packages reducen el acoplamiento entre contrato HTTP y host de aplicación.

### Inferencia fuerte

La dirección de la arquitectura es mover el boundary contractual fuera de `packages/opencode`, pero `dev` muestra una migración incremental, no un corte atómico.

---

## 5. Arquitectura histórica Hono

Branches antiguas como `refactor/node-server-adapter` sí contienen una generación donde `packages/opencode/src/server/server.ts` construía un `Hono` y concentraba routing, middleware, OpenAPI y listener concerns.

Esa versión debe etiquetarse **histórica**.

La comparación correcta es:

```text
HISTÓRICO
Hono singleton/app monolítica
        |
        v
MIGRACIÓN
bridges + HttpApi groups
        |
        v
DEV ACTUAL
Effect/Node host en packages/opencode
+ contracts/handlers parcialmente extraídos
```

La coincidencia del path `packages/opencode/src/server/server.ts` entre generaciones no implica que el contenido/arquitectura actual siga siendo Hono.

---

## 6. Boundary lógico frente a transporte físico

`Server.Default().app.fetch` demuestra en el baseline que el application boundary puede cruzarse in-process sin socket físico. `Server.listen()` materializa el mismo handler sobre HTTP.

Por tanto:

```text
application semantics != socket TCP
```

El contrato request/response/stream puede reutilizarse en topologías embebidas o en listener real.

---

## 7. Middleware y scoping

El composition root actual aplica capas específicas para:

- authorization;
- workspace routing;
- instance context;
- schema errors;
- CORS;
- compression/error/fence y otras capas según route group.

Esto confirma que location/workspace/session routing son concerns de frontera y no lógica que cada handler deba reconstruir individualmente.

---

## 8. Special transports

### SSE

Los eventos globales extraídos usan SSE. `packages/server/src/handlers/event.ts` crea una suscripción acotada, emite `server.connected` y heartbeat.

### PTY WebSocket

PTY conserva upgrade WebSocket/ticket-aware auth como excepción bidireccional específica.

### WebSocket RPC

Las branches de RPC WebSocket son experimentales/aditivas. No son el transport universal vigente de `dev`.

---

## 9. Process lifecycle fuera del API domain

Daemon/desktop supervisan procesos, discovery, health, auth y shutdown. El HTTP API no debe confundirse con ese lifecycle de proceso.

Este boundary sí es estable conceptualmente:

```text
process supervisor -> starts/discovers server
client             -> consumes API
server host         -> composes domain services
```

---

## 10. Qué está consolidado y qué sigue en transición

### Consolidado

- listener Effect/Node;
- handler in-process;
- HttpApi declarativo;
- contracts y handlers extraídos en packages dedicados;
- auth/routing middleware;
- SSE global y PTY WebSocket;
- composición mediante Effect layers/nodes.

### En transición

- ownership completo de todos los route groups;
- división `packages/opencode` ↔ `packages/server`;
- coexistencia de services legacy-compatible y V2;
- clientes/surfaces que aún dependen de compatibilidad histórica.

## Conclusión

El boundary actual no puede reducirse a `packages/server`. En `dev`, **`packages/opencode` sigue siendo el host de aplicación**, mientras `packages/protocol` y `packages/server` constituyen la extracción progresiva del contrato y de partes de la implementación. Esa formulación coincide con el composition root real y evita confundir la arquitectura objetivo con la ya ejecutada.