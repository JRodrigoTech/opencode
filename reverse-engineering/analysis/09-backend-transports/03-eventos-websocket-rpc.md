# 03 — Eventos, SSE, WebSocket y RPC

## 1. Modelo vigente: HTTP para commands/queries, streaming especializado

La arquitectura `dev` no adopta un único transport bidireccional universal. Usa:

- HTTP request/response para la mayor parte del API;
- SSE para eventos globales y eventos durables de sesión;
- WebSocket para PTY;
- un experimento separado (`websocket-rpc`) para evaluar RPC multiplexado.

Esta separación es arquitectónicamente importante: cada transport se usa donde encaja con la semántica del flujo.

---

## 2. Stream global: `GET /api/event`

### Contrato

`packages/protocol/src/groups/event.ts` declara `event.subscribe` como un `StreamSse`.

### Handler en `dev`

`packages/server/src/handlers/event.ts`:

1. obtiene `EventV2.Service`;
2. crea un stream bounded con capacidad 256;
3. antepone un evento sintético `server.connected`;
4. codifica cada evento como SSE `event: message` con JSON;
5. mezcla un heartbeat cada 15 segundos;
6. responde con `text/event-stream` y headers para desactivar caching/buffering.

Headers relevantes:

- `Cache-Control: no-cache, no-transform`;
- `X-Accel-Buffering: no`;
- `X-Content-Type-Options: nosniff`.

### Semántica

`server.connected` permite al consumidor distinguir “el stream ya está adquirido y observable” de la simple apertura TCP/HTTP.

El buffer bounded evita crecimiento sin límite si un subscriber consume más lentamente que la publicación.

### Inferencia

El stream global se diseña como canal de invalidación/actualización live, no como log durable completo. Para replay fuerte, la arquitectura introduce la superficie específica de sesión.

---

## 3. Evolución desde `event-subscribe`

### Hecho confirmado

En branch `event-subscribe`, `packages/server/src/handlers/event.ts` ya contiene:

- capacity 256;
- `server.connected`;
- heartbeat de 15 s;
- SSE;
- `EventV2.liveBounded(..., accept: isOpenCodeEvent)`.

En `dev`, el handler usa `EventV2.allBounded` y la selección de superficie pública ya queda respaldada por el contrato/manifiesto de eventos del servidor.

### Interpretación

La evolución mueve filtrado desde una decisión local del transport a una definición más central de qué eventos forman parte de la superficie pública.

---

## 4. Stream durable de sesión

`packages/protocol/src/groups/session.ts` añade dos operaciones complementarias:

### History

`GET /api/session/:sessionID/history`

- devuelve página finita de `SessionEvent.Durable`;
- acepta `after` y `limit`;
- devuelve `hasMore`.

### Live + replay

`GET /api/session/:sessionID/event`

- SSE;
- acepta `after` como aggregate sequence;
- reproduce eventos durables posteriores a esa sequence;
- continúa con eventos nuevos.

### Consecuencia arquitectónica

El cliente dispone de un patrón de resincronización robusto:

```text
lastSequence
    │
    ├─ history(after=lastSequence)  -> catch-up finito
    │
    └─ event(after=lastSequence)    -> replay + live
```

Esto es conceptualmente distinto del stream global, porque el cursor de sesión está anclado a una secuencia durable del agregado.

---

## 5. WebSocket vigente: PTY

### Contrato

`packages/protocol/src/groups/pty.ts` marca `GET /api/pty/:ptyID/connect` con metadata `x-websocket`.

### Flujo de autorización recomendado

1. cliente autenticado hace `POST /api/pty/:ptyID/connect-token`;
2. debe incluir `x-opencode-ticket: 1`, forzando preflight CORS en browser;
3. el servidor valida origin y existencia del PTY;
4. emite ticket corto de un solo uso, scoped a PTY/location;
5. cliente abre WebSocket a `/api/pty/:ptyID/connect?ticket=...`;
6. handler consume el ticket y sólo entonces acepta el stream.

### Orden y replay

`packages/server/src/handlers/pty.ts` usa una única outbox queue para serializar:

- replay inicial;
- metadata/cursor;
- output live;
- close frame.

El handler adjunta al PTY con cursor, activa live tras encolar replay y garantiza `detach()` al finalizar.

### Errores de socket

Sesión inexistente o ya terminada se representa con close code 4404 después del upgrade cuando corresponde.

### Boundary de seguridad

La excepción del middleware Basic Auth para un URL con ticket no elimina autenticación: cambia a una capability de mínimo alcance y corta duración.

---

## 6. `websocket-rpc`: alternativa experimental

### Evidencia

Branch `websocket-rpc`, commit `b2763569860285cc555ba401e3d4bf55e2f5702f` (2026-08-27).

La branch introduce un RPC derivado de los contratos existentes de `HttpApi` y una ruta WebSocket capaz de transportar requests tipadas.

### Decisiones confirmadas

- schemas RPC derivados del `HttpApi`, evitando duplicar contratos manualmente;
- un WebSocket scoped sirve múltiples operaciones;
- soporta operaciones unary y streams;
- multiplexa requests/responses mediante identidad de llamada;
- PTY sigue fuera del RPC genérico;
- el cliente no debe reejecutar ciegamente mutaciones si cae la conexión.

### Razón del último punto

Si el servidor procesó una mutación pero el cliente perdió el response, un retry automático sobre reconnect puede duplicar efectos. La branch prefiere propagar incertidumbre/fallo antes que introducir semántica at-least-once implícita.

### Comparación con `dev`

| Tema | `dev` | `websocket-rpc` |
| --- | --- | --- |
| Commands/queries | HTTP | RPC multiplexado sobre WS |
| Global events | SSE | stream RPC posible |
| Session events | SSE | stream RPC posible |
| PTY | WebSocket dedicado | permanece dedicado |
| Reconnect | a nivel de cliente/stream | debe reconstruir socket/scopes |
| Mutations in-flight | semántica HTTP normal | no replay automático |
| Contrato | HttpApi | derivado de HttpApi |

### Inferencia

La branch no intenta reemplazar el domain API, sino reemplazar/añadir una materialización del mismo contrato. Es evidencia adicional de que el boundary estable es el protocolo tipado, no el transport físico.

---

## 7. Branches `websocket-*` que no son backend transports

El patrón nominal produce falsos positivos importantes.

Branches como `websocket-auth`, `openai-websocket` y cambios similares pueden pertenecer al transporte **saliente** hacia proveedores LLM. `native-http-middleware` también termina en una línea de transport HTTP del stack AI/provider.

Estos cambios afectan cómo OpenCode habla con proveedores externos, no cómo UI/CLI/client hablan con el backend OpenCode.

Se catalogan en `06-inventario-branches.md` pero no se usan como evidencia del server architecture.

---

## 8. Sincronización: eventos vs proyecciones

`app-backend-v2` muestra por qué el stream no siempre contiene la representación UI completa.

En el test del commit `712d4fb8003be143a39c6af9d9efe980184da7d1`:

1. llega por `/api/event` un fragmento `session.text.delta`;
2. el adapter identifica la sesión/mensaje afectado;
3. recupera `/api/session/:sessionID/message/:messageID`;
4. emite a la app una `timeline.updated` con la proyección normalizada.

### Conclusión

El evento puede actuar como señal incremental/durable, mientras la proyección HTTP actúa como representación canónica para el consumer. No todo cliente está obligado a reconstruir toda la UI aplicando deltas de bajo nivel.

---

## 9. Fallos y backpressure

### SSE global

- buffer bounded;
- heartbeat mantiene intermediarios/conexión visibles;
- reconexión queda en consumer.

### Session durable SSE

- sequence permite replay después de desconexión;
- history permite catch-up controlado.

### PTY WebSocket

- queue única preserva ordering;
- cursor soporta replay;
- lifecycle de attachment se libera al cerrar.

### RPC experimental

- multiplexación reduce sockets;
- disconnect introduce incertidumbre para mutaciones y exige política explícita.

---

## 10. Hechos vs inferencias

### Confirmado

La combinación HTTP + SSE + PTY WebSocket existe en `dev`; el RPC WebSocket universal existe en una branch experimental separada.

### Inferencia

La preferencia de `dev` por transports especializados parece deliberada: conserva semántica HTTP sencilla para commands/queries y usa conexiones persistentes sólo donde replay, low-latency streaming o bidireccionalidad lo justifican.