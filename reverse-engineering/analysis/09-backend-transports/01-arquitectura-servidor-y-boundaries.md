# 01 — Arquitectura del servidor y boundaries

## 1. Baseline vigente en `dev`

### Hecho confirmado

La arquitectura vigente separa tres responsabilidades que antes estaban mezcladas:

1. **Contrato de transporte** en `packages/protocol`.
2. **Implementación del servidor** en `packages/server`.
3. **Consumers** en CLI, desktop y clients generados/adaptadores.

El boundary central ya no es una clase o singleton `Server` en `packages/opencode`, sino un contrato `HttpApi` compuesto de grupos tipados.

### Contrato

`packages/protocol/src/api.ts` agrega grupos funcionales. Los grupos contienen:

- método y path HTTP;
- schema de query, path params y payload;
- schema de success/error;
- middleware asociado;
- metadata OpenAPI;
- en casos especiales, semántica de stream SSE o upgrade WebSocket.

Ejemplos relevantes:

- `HealthGroup`: `GET /api/health`.
- `EventGroup`: `GET /api/event`, declarado como `StreamSse`.
- `makeSessionGroup(...)`: CRUD y comandos de sesión, history y event stream.
- `PtyGroup`: HTTP para lifecycle de PTY y WebSocket para connect.

### Implementación

`packages/server/src/api.ts` liga el `HttpApi` de protocol con middleware de servidor.

`packages/server/src/handlers.ts` y `packages/server/src/handlers/*` registran implementaciones mediante `HttpApiBuilder.group`.

`packages/server/src/routes.ts` compone:

- handlers tipados;
- middleware de auth;
- CORS;
- observability;
- services de dominio;
- OpenAPI;
- fallback/not-found.

### Inferencia

El diseño persigue que `protocol` sea un boundary estable que pueda ser consumido por varios runtimes sin conocer detalles del proceso que hospeda el servidor. La evidencia fuerte es que los mismos contratos sirven para listener HTTP, client generation y experimentos RPC/embedded.

---

## 2. Arquitectura anterior: servidor Hono dentro de `packages/opencode`

### Hecho confirmado

En ramas antiguas como `refactor/node-server-adapter` el servidor principal vive en:

`packages/opencode/src/server/server.ts`

La función `Server.createApp(...)` construye un `Hono` y concentra:

- logging;
- CORS;
- Basic Auth;
- routing global e instance-scoped;
- `GET /openapi`;
- error handling;
- WebSocket upgrade injection;
- health y event routing;
- listener startup con `Bun.serve` o adapter Node.

La misma unidad expone `Server.Default()` y `Server.listen(...)`.

### Diferencia estructural con `dev`

Antes:

```text
Server singleton
  ├─ Hono app
  ├─ middleware
  ├─ routing
  ├─ instance context
  ├─ SSE / websocket glue
  ├─ listener choice
  └─ OpenAPI
```

Ahora:

```text
protocol.HttpApi
  ├─ contracts
  └─ schemas
       │
server
  ├─ handlers
  ├─ auth/cors/location middleware
  └─ listener/routes composition
       │
clients / desktop / cli
```

### Interpretación evolutiva

La separación actual reduce tres acoplamientos históricos:

1. contrato ↔ framework Hono;
2. routing ↔ process listener;
3. SDK ↔ estructura interna del servidor.

---

## 3. Migración Hono → Effect HttpApi

### Evidencia directa

La familia `kit/httpapi-*` contiene una migración incremental explícita. El commit `2b028287e21ae5931d6654738872c7b5ba9bc55a` (`kit/httpapi-route-inventory-current`) añade un script que inspecciona las rutas registradas de Hono y las clasifica como:

- `bridged`;
- `next`;
- `later`;
- `special`.

El inventario marca `event`, `pty` y `tui` como especiales por sus transports/semánticas.

### Hecho confirmado

No se intentó reescribir todo de una vez. La estrategia fue:

1. mantener Hono como surface existente;
2. implementar grupos `HttpApi` para subconjuntos;
3. montar un bridge;
4. conservar fallback hacia rutas Hono;
5. generar/inventariar paridad;
6. mover progresivamente rutas al nuevo contrato.

El commit `f346024f0e0d1d98bfa83c8f1216bf988af2abc1` en `backend-adapter-v2` corrige precisamente el sharing del router entre rutas API y fallback, evidencia de coexistencia durante la transición.

### Resultado en `dev`

La arquitectura que era un bridge incremental pasa a ser la estructura primaria: `packages/protocol` y `packages/server` son paquetes separados.

---

## 4. Boundary de transporte: lógico frente a físico

### Hecho confirmado

La arquitectura admite al menos tres formas de atravesar el mismo boundary conceptual:

1. HTTP sobre TCP hacia listener local/remoto.
2. Un `fetch` in-process contra un server embebido.
3. En `websocket-rpc`, RPC derivado del mismo contrato sobre un WebSocket.

`embedded-server-dispose` encapsula un server in-process y construye un `fetch` que llama `server.app.fetch(...)`. El cliente sigue usando una URL sintética (`http://opencode.internal`) aunque no haya socket físico.

### Inferencia fuerte

El verdadero boundary del backend es **request/response/stream semantics + schemas**, no “HTTP como red”. El transporte de red es una materialización del contrato.

Esto explica por qué:

- el desktop puede tratar un sidecar como servidor remoto aunque viva en la misma máquina;
- el CLI puede usar un server embebido;
- un RPC WebSocket puede derivarse del `HttpApi` sin reescribir el dominio.

---

## 5. Boundaries internos observables

### 5.1 Listener / application

El listener es reemplazable. Históricamente hubo Bun y Node adapters; `refactor/node-server-adapter` añade `@hono/node-server` sin cambiar el resto del routing.

**Conclusión:** el listener no es el backend domain boundary.

### 5.2 Protocol / implementation

Los grupos de `protocol` no dependen de core implementations; los handlers sí.

**Conclusión:** éste es un boundary contractual real.

### 5.3 Location / session

Muchas APIs trabajan bajo un location context, pero las sesiones además tienen middleware de resolución propio.

**Conclusión:** el servidor no es un simple CRUD global; existe scoping explícito por directory/workspace/session.

### 5.4 Process supervisor / API server

El daemon/desktop supervisor decide cómo arrancar, descubrir, autenticar y detener el servidor, pero el API se consume por la misma superficie.

**Conclusión:** process lifecycle está deliberadamente fuera del core del HTTP API.

### 5.5 Special transports

SSE y PTY WebSocket se modelan como excepciones puntuales, no como un transport universal.

**Conclusión:** `dev` favorece HTTP convencional para commands/queries y transports streaming únicamente donde aportan semántica necesaria.

---

## 6. Qué llegó a `dev` y qué quedó atrás

### Llegó a `dev`

- contratos declarativos con schemas;
- OpenAPI derivado del contrato;
- separación protocol/server;
- middleware tipado;
- SSE global;
- stream durable por sesión;
- WebSocket específico para PTY;
- auth del listener;
- sidecar/daemon supervisado fuera del server package;
- capacidad de abstraer el backend detrás de client/adapters.

### Fue sustituido o no es baseline

- singleton Hono como arquitectura central;
- bridge Hono/Effect como mecanismo permanente;
- RPC WebSocket universal;
- managed-service client API extensa tal como existía en `client-service`;
- elección de listener Bun/Node dentro del antiguo singleton Server como preocupación de aplicación.

## 7. Hechos vs hipótesis

### Confirmado

La estructura de paquetes, contratos, handlers, SSE, PTY WebSocket, daemon y desktop sidecar descrita aquí existe en código o commits inspeccionados.

### Hipótesis

Que el objetivo explícito de la separación fuese soportar múltiples transports futuros no aparece como una única declaración normativa. Sin embargo, la coexistencia de HttpApi, embedded fetch y WebSocket RPC derivado del contrato hace esa interpretación altamente consistente con la evidencia.