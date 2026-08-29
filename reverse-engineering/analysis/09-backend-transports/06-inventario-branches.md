# 06 — Inventario y clasificación de branches

## 1. Criterio de inclusión

La búsqueda se realizó por familias nominales solicitadas (`server`, `service`, `daemon`, `websocket`, `backend`, `client-service`, `event-subscribe`, `native-http`, `httpapi`, además de RPC/discovery/embedded).

Una coincidencia de nombre no se considera automáticamente evidencia del backend. Cada familia se clasifica como:

- **núcleo**: modifica directamente server architecture, lifecycle, discovery o transport cliente↔backend;
- **migración/paridad**: participa en la transición arquitectónica sin introducir un transport nuevo;
- **adyacente**: toca server/client pero su foco primario es otro subsistema;
- **falso positivo nominal**: el nombre coincide, pero el transport es por ejemplo OpenCode↔provider LLM o un internal Effect service.

## 2. Familia `server-*` y equivalentes

Branches encontradas por búsqueda `server`:

- `server-auto-approve`
- `server-discovery`
- `server-location-paths`
- `server-node-graph`
- `server-profiles`
- `server-switcher`
- `brendan/node-server-types`
- `browser-server`
- `cleanup-server-routes`
- `copilot/research-opencode-server-plugin-api`
- `daxbox-servers`
- `disable-auto-server`
- `embedded-server-dispose`
- `jlongster/remove-workspace-server`
- `kit/server-listen-native`
- `nxl/mount-question-server`
- `opencode/permission-server-fallback`
- `otel-server-shutdown`
- `publish-server`
- `refactor/node-server-adapter`
- `refactor/server-route-organization`
- `skill-source-observer`
- `snapshot-spawn-server-logs`
- `test/processor-mock-server`

### Núcleo/hitos

#### `server-discovery`

Primera generación relevante de discovery local mediante registro URL/PID + health. Precursor directo del service/daemon registry.

#### `refactor/node-server-adapter`

Evidencia de la arquitectura Hono monolítica y de la separación incipiente entre app HTTP y listener runtime (Bun/Node).

#### `cleanup-server-routes` / `refactor/server-route-organization`

Pertenecen a la reorganización de la surface Hono previa a la separación actual. Se agrupan como refactor de routing, no como transports distintos.

#### `embedded-server-dispose`

Hito del backend embebido: `fetch` in-process + lifecycle/dispose explícito.

#### `kit/server-listen-native`

Adyacente a la abstracción de listener; relevante como evidencia de que listener/runtime es sustituible, pero no altera el domain protocol por sí solo.

#### `otel-server-shutdown`

Adyacente al lifecycle: observability/flush durante teardown. Su principio aparece también en `embedded-server-dispose`.

### Adyacentes/no centrales

Branches como `server-auto-approve`, `server-location-paths`, `nxl/mount-question-server` o `opencode/permission-server-fallback` tocan server routes o comportamiento de subsistemas concretos. Son útiles para esas áreas, pero no constituyen una generación independiente de backend/transport.

Branches de test/research (`test/processor-mock-server`, `copilot/research-opencode-server-plugin-api`) no se usan como fuente primaria del linaje runtime.

---

## 3. Familia `service-*`

Branches encontradas:

- `service-channel-config`
- `service-election-safety`
- `service-errors`
- `service-probe-timeout`
- `service-restart`
- `service-shutdown-watchdog`
- `service-status`
- `service-test-timeout`
- `service-version-guard`
- `authenticated-service-stop`
- `client-service`
- `codemode-service`
- `effect/config-paths-service`
- `effect/define-service-helper`
- `effect/session-transport-service`
- `effect-sync-event-service`
- `feat/core-config-service`
- `kit/instance-loader-service`
- `kit/shell-job-service`
- `llm-service-event-seam`
- `nxl/fff-search-service`
- `nxl/runtime-aware-search-service`
- `plugin-service-tree`
- `stale-service-version`
- `storage-v2-service`
- `thdxr/auth-well-known-service`
- `validate-service-data`
- `websearch-consent-service`
- `worktree-audit-effect-services`

### Línea evolutiva principal

Se agrupan como una única generación de **managed local service**:

- `client-service`
- `service-election-safety`
- `service-channel-config`
- `service-errors`
- `service-probe-timeout`
- `service-status`
- `service-restart`
- `service-version-guard`
- `stale-service-version`
- `validate-service-data`
- `authenticated-service-stop`
- `service-shutdown-watchdog`
- `service-test-timeout` (test hardening, no nueva arquitectura)

### Hitos dentro de la línea

- `client-service`: discovery/ensure/stop como API de cliente y registration como contrato completo.
- `service-election-safety`: refuerza exclusión/ownership; conceptualmente solapa `daemon-election`.
- `service-probe-timeout`: acota detección de incumbents no responsivos.
- `service-status`: hace explícitos estados de startup/readiness/failure.
- `service-restart`: separa restart deliberado de reconnect automático.
- `service-version-guard`: negociación direccional y protección frente a cliente antiguo/server nuevo.
- `authenticated-service-stop`: exige prueba HTTP autenticada antes de shutdown destructivo.
- `service-shutdown-watchdog`: límite externo al teardown cooperativo.

### Falsos positivos/internal services

No pertenecen al daemon/backend transport por el mero sufijo `service`:

- `codemode-service`
- `effect/config-paths-service`
- `effect/define-service-helper`
- `effect/session-transport-service` — relevante para boundaries internos de Effect, no equivalente al daemon local sin evidencia adicional.
- `effect-sync-event-service`
- `feat/core-config-service`
- `kit/instance-loader-service`
- `kit/shell-job-service`
- `llm-service-event-seam`
- `nxl/fff-search-service`
- `nxl/runtime-aware-search-service`
- `plugin-service-tree`
- `storage-v2-service`
- `thdxr/auth-well-known-service`
- `websearch-consent-service`
- `worktree-audit-effect-services`

Estas ramas deben cruzarse con AGENT 10 (Effect/services) o con los subsistemas correspondientes, no atribuirse a la evolución del daemon.

---

## 4. Familia `daemon-*`

Branches encontradas:

- `daemon-election`
- `desktop-v2-daemon`
- `models-refresh-daemon`

### `daemon-election`

Núcleo. Election/lock ownership y failure modes de procesos locales.

### `desktop-v2-daemon`

Núcleo. Integra la generación daemon/service con el desktop y constituye el antecedente del path sidecar v2 visible en `dev`.

### `models-refresh-daemon`

Falso positivo para este análisis: “daemon” describe una tarea de refresh de modelos, no el backend server process compartido.

---

## 5. Familia `websocket-*`

Branches encontradas:

- `websocket-auth`
- `websocket-rpc`
- `websocket-upgrade-diagnostics`
- `desktop-websocket`
- `fix/openai-websocket-header-timeout`
- `openai-websocket`

### `websocket-rpc`

Núcleo y generación propia. Propone RPC tipado multiplexado sobre un único WebSocket derivado del `HttpApi`.

### `desktop-websocket`

Relevante como exploración del canal desktop/backend; debe interpretarse dentro de la línea de comunicación desktop, no asumirse automáticamente como arquitectura vigente. `dev` actualmente mantiene HTTP/SSE y WebSocket PTY especializado.

### Provider/outbound false positives

- `websocket-auth`
- `fix/openai-websocket-header-timeout`
- `openai-websocket`
- parte de `websocket-upgrade-diagnostics` según el código afectado

Estas ramas pertenecen al transporte OpenCode↔provider/model o a diagnóstico de upgrades, y por tanto corresponden principalmente al análisis de providers/AI stack.

---

## 6. Familia `backend-*`

Branches encontradas:

- `backend-adapter-v2`
- `app-backend-v2`
- `jlongster/fuzz-backend`

### `app-backend-v2`

Núcleo para el boundary UI↔API. Introduce/explicita un adapter que normaliza el API v2 hacia `AppClient` y transforma eventos/proyecciones.

### `backend-adapter-v2`

Núcleo de migración. Contiene coexistencia HttpApi/fallback y fixes de router compartido durante transición.

### `jlongster/fuzz-backend`

Testing/fuzzing; útil para calidad pero no una arquitectura alternativa por sí misma.

---

## 7. `event-subscribe`

Única branch con coincidencia exacta.

Núcleo de la evolución SSE. Ya contiene el patrón que llega a `dev`:

- bounded subscription;
- `server.connected`;
- heartbeat;
- SSE encoding.

La diferencia principal observada es el filtrado explícito `isOpenCodeEvent` en esa generación frente a la selección más centralizada actual.

---

## 8. Familia `native-http-*`

Encontrada:

- `native-http-middleware`

### Clasificación

Falso positivo para backend cliente↔servidor como línea principal. Su tip commit y diff están ligados a `packages/ai` y preservación de transforms de requests del transport hacia providers.

Debe analizarse con AGENT 06 (providers/LLM transports), no confundirse con la migración Hono→Effect del servidor OpenCode.

---

## 9. Familia `httpapi-*`

Branches encontradas:

- `fix/httpapi-query-schema-drift`
- `kit/config-httpapi`
- `kit/config-providers-httpapi-spike`
- `kit/file-httpapi-spike`
- `kit/fix-httpapi-session-list-main`
- `kit/httpapi-auth-cleanup-base`
- `kit/httpapi-exercise-via-sdk`
- `kit/httpapi-experimental-tools`
- `kit/httpapi-json-shape-parity`
- `kit/httpapi-listener-proxy-ws`
- `kit/httpapi-not-found-shape`
- `kit/httpapi-project-skill-repro`
- `kit/httpapi-route-inventory`
- `kit/httpapi-route-inventory-current`
- `kit/httpapi-route-parity`
- `kit/httpapi-sdk-smoke-test`
- `kit/httpapi-total-coverage`
- `kit/httpapi-workspace-schema-migration`
- `kit/project-httpapi-reads`
- `kit/question-httpapi-spike`
- `kit/workspace-httpapi-reads`

### Agrupación

Estas ramas forman una campaña evolutiva coherente de **migración Hono → Effect HttpApi**.

No deben documentarse como 21 diseños alternativos. Sus roles son:

#### Spikes/port por dominio

- `kit/config-httpapi`
- `kit/config-providers-httpapi-spike`
- `kit/file-httpapi-spike`
- `kit/project-httpapi-reads`
- `kit/question-httpapi-spike`
- `kit/workspace-httpapi-reads`
- `kit/httpapi-workspace-schema-migration`
- `kit/httpapi-experimental-tools`

#### Paridad y shape compatibility

- `fix/httpapi-query-schema-drift`
- `kit/httpapi-json-shape-parity`
- `kit/httpapi-not-found-shape`
- `kit/httpapi-route-parity`
- `kit/fix-httpapi-session-list-main`

#### Inventario y coverage

- `kit/httpapi-route-inventory`
- `kit/httpapi-route-inventory-current`
- `kit/httpapi-total-coverage`

#### Auth/listener/transports especiales

- `kit/httpapi-auth-cleanup-base`
- `kit/httpapi-listener-proxy-ws`

#### SDK/exercise

- `kit/httpapi-exercise-via-sdk`
- `kit/httpapi-sdk-smoke-test`
- `kit/httpapi-project-skill-repro`

### Hito representativo

`kit/httpapi-route-inventory-current` (`2b028287e21ae5931d6654738872c7b5ba9bc55a`) documenta explícitamente la estrategia bridged/next/later/special y por ello es una de las mejores snapshots del estado intermedio.

---

## 10. Otras branches equivalentes por concepto

### RPC

`websocket-rpc` es la rama materialmente relevante detectada para RPC genérico. No se observó en `dev` una sustitución del API general por RPC.

### Discovery

`server-discovery` → `client-service`/`service-*` → `Daemon` actual forman el linaje principal.

### Embedded

`embedded-server-dispose` es el hito principal inspeccionado.

### Desktop/backend

`desktop-v2-daemon`, `app-backend-v2`, `backend-adapter-v2` forman una familia de transición en la que UI y desktop dejan de depender de un backend v1 específico y consumen una abstracción estable.

---

## 11. Matriz de conceptos que llegaron a `dev`

| Concepto histórico | Evidencia histórica | Estado en `dev` |
| --- | --- | --- |
| discovery por registro | `server-discovery` | sí, evolucionado |
| auth de daemon local | `client-service` | sí |
| health como identity gate | `client-service` | sí |
| election safety | `daemon-election` | principio conservado, implementación cambiada |
| version-aware lifecycle | `service-version-guard` | parcialmente/conceptualmente; no asumir API idéntica |
| authenticated stop | `authenticated-service-stop` | auth/PID-safety conservada; política exacta difiere |
| restart explícito | `service-restart` | lifecycle existe, API histórica no necesariamente |
| shutdown bounded | `service-shutdown-watchdog` | principio de teardown robusto visible en varios paths |
| Hono→HttpApi | `kit/httpapi-*` | sí, consolidado |
| SSE bounded + heartbeat | `event-subscribe` | sí |
| session durable replay | evolución posterior HttpApi/session | sí |
| PTY WebSocket dedicado | ramas/migración HttpApi | sí |
| generic WebSocket RPC | `websocket-rpc` | no como default |
| app backend adapter | `app-backend-v2` | sí como principio de desacoplamiento |
| embedded fetch server | `embedded-server-dispose` | arquitectura compatible; no confundir con listener default |

---

## 12. Ideas descartadas o no consolidadas como baseline

### No default en `dev`

- RPC genérico sobre un único WebSocket.
- API extensa `Service.discover/incumbent/ensure/restart/...` exactamente como en `client-service`.
- Hono como fuente de verdad de rutas.
- bridge permanente Hono↔HttpApi.
- tratar cualquier PID registrado como suficiente para control destructivo.

### Sustituidas por una forma posterior

- `server.json` URL/PID sin identidad fuerte → registro autenticado/health validation.
- filtrado SSE exclusivamente en handler → superficie de eventos más centralizada.
- sidecar desktop único y totalmente owned → coexistencia/migración hacia background CLI/daemon.

---

## 13. Relación evolutiva resumida

```text
Hono Server
 ├─ refactor/server-route-organization
 ├─ refactor/node-server-adapter
 └─ kit/httpapi-* migration
          │
          ▼
 protocol HttpApi + server handlers (dev)
          │
          ├─ SSE global/session
          ├─ PTY WebSocket
          └─ websocket-rpc (experimental alternate transport)

server-discovery
      │
      ▼
daemon-election + client-service
      │
      ├─ service-status/probe/election safety
      ├─ service-restart
      ├─ service-version-guard
      ├─ authenticated-service-stop
      └─ service-shutdown-watchdog
      │
      ▼
CLI Daemon in dev
      │
      └─ desktop-v2-daemon / current background CLI path

app-backend-v2 + backend-adapter-v2
      │
      ▼
UI/client normalization + stable backend boundary
```

## 14. Nota de exhaustividad

El inventario es exhaustivo respecto a las coincidencias devueltas por las búsquedas nominales realizadas en el repositorio para las familias indicadas. La clasificación evita afirmar que una branch es parte del backend sólo por su nombre; cuando la evidencia inspeccionada muestra que pertenece a provider transports, internal Effect services, tests o features de dominio, se indica expresamente.