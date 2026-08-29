# 05 — Desktop, embedded server y sincronización cliente↔backend

## 1. Desktop actual: Electron main como supervisor

`packages/desktop/src/main/index.ts` deja una separación clara:

- Electron main administra procesos y credenciales;
- el sidecar hospeda el backend;
- el renderer recibe un endpoint ya inicializado mediante IPC;
- el renderer se comporta como cliente del API, no como owner del proceso.

El contrato IPC relevante es `ServerReadyData` en `packages/desktop/src/preload/types.ts`:

```ts
{
  url: string
  username: string | null
  password: string | null
}
```

## 2. Dos generaciones de sidecar visibles en `dev`

`SIDECAR_VERSION` selecciona por `OPENCODE_SIDECAR_V2`.

### v2

Electron llama `startBackgroundCli(...)` y recibe:

- URL;
- username;
- password.

Una vez disponibles, resuelve `serverReady`.

### legacy/v1

Electron:

1. elige/reserva un puerto loopback;
2. usa `127.0.0.1`;
3. genera password aleatorio;
4. lanza `spawnLocalServer(hostname, port, password, ...)`;
5. publica `{ url, username: "opencode", password }` al renderer.

### Conclusión

El frontend no necesita saber si el backend procede de un utility process legacy o de un background CLI/daemon moderno. Ambos convergen en el mismo endpoint+credentials boundary.

---

## 3. Lifecycle del sidecar Electron

`packages/desktop/src/main/server.ts` usa `utilityProcess.fork(sidecar.js)`.

### Startup protocol

Electron envía al child:

```text
{ type: "start", hostname, port, password, userDataPath }
```

El sidecar responde mediante IPC de proceso con:

- `ready`;
- `stopped`;
- `error`.

Existe un timeout de stall de 60 segundos para llegar a `ready`.

Después se realiza health polling contra:

- `/api/health`;
- fallback `/global/health` para compatibilidad.

Si hay password se añade Basic Auth `opencode:<password>`.

### Shutdown

`stop()`:

1. envía `{ type: "stop" }`;
2. espera la salida;
3. después de 6 segundos puede matar el utility process.

El desktop ejecuta stop de sidecars en:

- `before-quit`;
- `will-quit`;
- relaunch;
- `SIGINT`;
- `SIGTERM`.

También coordina sidecars WSL en Windows.

### Networking local

El desktop configura explícitamente bypass de proxy para loopback (`127.0.0.1`, `localhost`, `::1`) y el switch de Electron correspondiente. Esto evita que una política proxy del usuario intercepte accidentalmente la comunicación con el backend local.

---

## 4. Renderer communication boundary

`registerIpcHandlers` expone `awaitInitialization()`.

El renderer no obtiene URL/credentials hasta que el Deferred `serverReady` se resuelve.

### Consecuencia

Startup de UI y startup de backend están sincronizados mediante una barrera explícita.

El renderer puede cargar shell/window state independientemente, pero las operaciones API necesitan el resultado de inicialización.

### Seguridad

Las credenciales del sidecar no se generan en la UI web. Electron main actúa como broker de secrets y lifecycle.

---

## 5. Branch `desktop-v2-daemon`

Esta familia representa el movimiento desde un sidecar estrictamente propiedad de una ventana/aplicación hacia un daemon/background service reutilizable por el desktop.

### Hecho confirmado

La branch existe como línea específica (`desktop-v2-daemon`) y el `dev` actual conserva un selector de sidecar `v1`/`v2`, donde v2 arranca el background CLI.

### Inferencia fuerte

El selector vigente es residuo explícito de una migración gradual: se preservó temporalmente el sidecar legacy mientras se validaba la nueva semántica daemon/service.

---

## 6. `app-backend-v2`: adapter entre protocolo y modelo de aplicación

Commit principal:

`712d4fb8003be143a39c6af9d9efe980184da7d1` — `feat(app): add v2 backend adapter`.

La función `createV2Backend(OpenCodeClient, defaultLocation)` devuelve un `AppClient` y actúa como anti-corruption layer entre el API v2 y las abstracciones de la aplicación.

### Normalización observada en tests

- pagina sesiones y transforma responses;
- aplica precedencia entre location explícita y default;
- normaliza respuestas binarias de filesystem;
- cambia agent/model antes de admitir prompt cuando la selección de UI lo requiere;
- encapsula stage revert;
- convierte eventos de protocolo a eventos de aplicación.

### Importancia

El desktop/app no acopla su state model directamente a cada DTO del servidor. El adapter puede cambiar la forma de sincronización sin obligar a reescribir componentes UI.

---

## 7. Sincronización por evento + refresh de proyección

Un test de `app-backend-v2` documenta un patrón clave.

Cuando el adapter recibe por `/api/event`:

```text
session.text.delta
```

no necesariamente propaga el delta crudo a la UI. Puede:

1. identificar `sessionID` y `assistantMessageID`;
2. llamar `GET /api/session/:sessionID/message/:messageID`;
3. convertir la respuesta a `TimelineItem`;
4. emitir `timeline.updated`.

### Ventaja

La UI consume una proyección coherente aunque el stream de dominio evolucione o contenga fragmentos de bajo nivel.

### Trade-off

Aumenta lecturas HTTP durante streaming, pero reduce lógica de replay/proyección en múltiples clientes.

---

## 8. Eventos durables y sincronización moderna

El API actual mejora el patrón anterior añadiendo:

- `session.history`;
- `session.events(after=sequence)`;
- `session.message`.

Esto permite combinar:

```text
SSE durable -> señala cambios ordenados
HTTP projection -> reconstruye estado concreto
history -> recupera gaps
```

### Inferencia

El sistema converge hacia CQRS ligero/event-driven projection sin exigir event sourcing completo en el cliente: el backend conserva autoridad sobre proyecciones y el cliente usa el log durable para orden/recovery.

---

## 9. Server embebido: el backend sin proceso de red

Branch `embedded-server-dispose`, commit:

`31d25c8f084dfe5ee35e855e1e40be4159d0799c`.

### Hecho confirmado

Se introduce helper `embeddedServer(auth?)` que:

1. importa `Server`;
2. obtiene `Server.Default()`;
3. construye un `fetch` que llama directamente `server.app.fetch(...)`;
4. inyecta `Authorization` si procede;
5. devuelve también `server.dispose`.

El cliente SDK sigue usando una base URL sintética (`http://opencode.internal`).

### Disposal

Los paths interactive y non-interactive ejecutan `dispose()` en `finally`.

El TUI worker también dispone el server default durante shutdown.

Un test comprueba que el disposal permite flush de telemetría OTLP (`/v1/traces`) antes de terminar el proceso.

### Conclusión arquitectónica

El servidor puede ser:

- process externo;
- Electron sidecar;
- daemon compartido;
- handler embebido in-process.

El client boundary permanece esencialmente igual.

---

## 10. Embedded server vs listener server

| Propiedad | Listener | Embedded |
| --- | --- | --- |
| TCP socket | sí | no |
| URL lógica | real | sintética posible |
| `Request`/`Response` | sí | sí |
| auth middleware | sí | puede inyectarse igual |
| HttpApi handlers | mismos conceptos | mismos conceptos |
| cleanup | cerrar listener + scopes | `dispose()` scopes |
| observability flush | parte de shutdown | requiere disposal explícito |

### Inferencia

La posibilidad embedded demuestra que la composición de Effect layers/resources pertenece al lifecycle del server application, no exclusivamente al listener TCP.

---

## 11. Restart y reconnect desde clientes

La branch `service-restart` muestra una regla importante para clientes con streams activos:

- abortar event stream actual;
- cancelar resolución/reconnect pendiente;
- ejecutar restart explícito;
- reabrir stream después.

Sin esta secuencia, el cliente puede conectar de nuevo al daemon viejo mientras el usuario intenta reemplazarlo, o mantener dos pipelines de eventos concurrentes.

---

## 12. WSL y multiplicidad de backends

El desktop actual contiene `createWslServersController` y `spawnWslSidecar`.

### Hecho confirmado

En Windows pueden coexistir sidecars por distro WSL además del sidecar local.

### Implicación

“Backend” no equivale a singleton global de la aplicación desktop. La capa UI necesita manejar endpoints/contextos que pueden vivir en entornos de ejecución diferentes.

Esto refuerza la utilidad del `ServerReadyData`/endpoint abstraction.

---

## 13. Transport boundaries finales

### Process boundary

Electron main ↔ sidecar:

- utilityProcess IPC para lifecycle;
- HTTP/SSE/WS para API data plane.

### Renderer boundary

Renderer ↔ Electron main:

- preload IPC para inicialización, filesystem/native capabilities y lifecycle privilegiado.

Renderer ↔ backend:

- API client sobre endpoint proporcionado por Electron.

### Embedded boundary

CLI/TUI ↔ backend:

- mismo modelo `fetch(Request)->Response`, sin red.

### Inferencia

La arquitectura separa control plane local (spawn/stop/credentials/native OS) del data plane de OpenCode (HttpApi/events). Es una división particularmente útil para desktop security y testability.

---

## 14. Hechos vs inferencias

### Confirmado

Los dos sidecar paths, IPC `ServerReadyData`, health checks, shutdown hooks, embedded fetch/dispose y normalizaciones del adapter aparecen directamente en código/commits.

### Inferencia

La lectura como “control plane vs data plane” no es un nombre utilizado necesariamente por el repositorio, pero describe con precisión los boundaries observados.