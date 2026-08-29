# Familia `v2` / `beta`

## Clasificación

**HECHO.** `v2` y `beta` son refs idénticas en el corte analizado (`106629aa118086be7def6123241a9bf056ba77b6`).

**HECHO.** El PR #41627 declara que `beta` se construye desde `v2`; #41626 publica desktop beta con el CLI V2.

Por tanto, este documento trata ambas ramas como una única arquitectura, usando `beta/v2` para referirse a ella.

## Propósito

V2 es una reestructuración transversal de OpenCode orientada a:

- separar dominio/runtime de transporte y UI;
- hacer explícitos los contratos durables de sesión;
- tratar eventos y proyecciones como fuente de sincronización/recovery;
- convertir tools, providers, plugins y config en servicios registrables y reemplazables;
- soportar múltiples runtimes (`bun`, `node`, `workerd`);
- producir clientes desde un `HttpApi` y no desde handlers ad hoc;
- reducir el rol del paquete monolítico `packages/opencode`.

La propia especificación V2 fija la autoridad:

```text
Protocol -> HTTP operations / transport errors
Schema   -> public shapes / durable events
Core     -> runtime / persistence
Server   -> HTTP delivery / event feed / subscriber lifecycle
```

## 1. Descomposición de paquetes

### `dev`

El comando principal del root continúa entrando por:

```text
packages/opencode/src/index.ts
```

El repositorio ya contiene `core`, `server` y `cli`, pero todavía conviven con la aplicación legacy central.

### `beta/v2`

El root cambia a:

```text
packages/cli/src/index.ts
```

`@opencode-ai/cli` depende de:

- `@opencode-ai/client`
- `@opencode-ai/server`
- `@opencode-ai/tui`
- `@opencode-ai/schema`
- `@opencode-ai/plugin`
- `@opencode-ai/util`

`@opencode-ai/server` depende de:

- `@opencode-ai/core`
- `@opencode-ai/protocol`
- `@opencode-ai/schema`

`@opencode-ai/core` contiene providers, session runner, database, tools, MCP, filesystem y abstracciones runtime.

### Lectura arquitectónica

**INFERENCIA fuerte.** El CLI deja de ser “un modo de ejecutar el monolito” y pasa a ser el composition root de servicios ya separados.

## 2. Runtime de sesión

La sesión V2 está construida alrededor de una separación explícita entre **admission** y **execution**.

### Durable inbox

`Session.prompt(...)` publica primero un hecho durable:

```text
session.inbox.enqueued
```

Su proyección crea un row `session_inbox`. Sólo cuando se entrega se publica/proyecta:

```text
session.inbox.delivered
```

El input pasa entonces a ser visible al modelo como mensaje.

Consecuencias:

- admitir input no implica ejecutar inmediatamente;
- `resume: false` permite durabilidad sin wake;
- retry de admission puede ser idempotente por ID;
- compaction/move usan el mismo canal como control items;
- la cola pendiente sobrevive a interrupciones de ejecución.

### Modos `steer` y `queue`

- `steer`: se entrega en el siguiente safe step boundary.
- `queue`: espera hasta una boundary idle cuando la sesión ya no puede continuar por sí misma.

Los steers tienen prioridad sobre queues y un control item crea una boundary que no puede ser atravesada por input posterior.

### Execution ownership

`SessionExecution` es process-global y keyed por Session ID.

`SessionRunCoordinator` establece:

- resumes explícitos se unen a una ejecución activa;
- wakes repetidos se coalescen;
- sesiones distintas ejecutan concurrentemente;
- interrupt sólo afecta ownership local;
- pending input no se borra por interrupt.

### Crash recovery

V2 escribe un execution claim antes del busy period.

- success/failure/user interruption libera el claim;
- shutdown no limpio puede dejarlo vivo;
- startup localiza claims y reanuda top-level sessions;
- running tools huérfanas se marcan failed antes de continuar;
- no se promete exactly-once para provider requests ni efectos externos.

Este diseño reconoce explícitamente el límite de recuperación distribuida: sin fencing/placement protocol, execution ownership continúa siendo process-local.

## 3. Step model, retry y continuation

Cada Step recarga:

- Session History;
- agent seleccionado;
- model;
- instructions;
- tools.

Un Step lógico puede incluir varios Physical Attempts.

Motivos:

- scheduled retry;
- provider continuation rejection;
- incomplete stream;
- overflow + compaction.

### Retry

El generic retry incluye:

- rate limits;
- provider internal errors;
- transport unsent/unknown delivery;
- incomplete streams.

Hay request inicial + hasta 4 retries con jittered exponential backoff.

Antes de output durable se conserva el logical step y assistant message ID. Si hay partial durable output, se preserva el assistant parcial y se crea una continuation instruction con nuevo assistant message ID.

## 4. Compaction

V2 no trata la compaction como “borrar mensajes”, sino como cambiar el active model history manteniendo el transcript durable completo.

Flujo:

```text
full durable transcript
      |
      +--> compaction boundary
              |
              +--> rolling summary
              +--> bounded recent tail
              |
              +--> new active context epoch
```

Automatic compaction evalúa request completo + output headroom contra model context.

Ante provider context overflow antes de output durable, V2 puede ejecutar una compaction reactiva y reconstruir el mismo logical Step una vez.

Manual compaction reutiliza el mismo checkpoint machinery pero no constituye provider turn.

## 5. Instructions

Las instrucciones son valores versionados por hash, no un string global mutable.

Evento:

```text
session.instructions.updated
```

Cada source key apunta a SHA-256 de su contenido; los blobs canónicos se guardan una sola vez.

Fuentes combinadas por el runner:

- built-ins;
- ambient discovery;
- skill guidance del agent;
- references;
- MCP guidance;
- API-managed entries.

Una compaction crea un nuevo instruction epoch. Los cambios posteriores se incorporan cronológicamente como System messages renderizados en el momento de admission.

## 6. Tools

V2 define un único valor estructural:

```text
Tool.make({ input, output, execute })
```

### Tres salidas posibles

Una ejecución puede devolver:

1. `output`: machine value schema-validated para Code Mode;
2. `content`: representación model-facing durable;
3. `metadata`: JSON compacto para UI.

Esto evita que una misma estructura tenga que satisfacer simultáneamente máquina, modelo y UI.

### Registro scoped

Los tools se registran por scope:

```text
latest registration wins
scope closes -> registration disappears
previous active registration becomes visible again
```

Cada model request captura un snapshot de registrations. Una llamada ejecuta exactamente el tool que fue anunciado en ese request, aunque el registry cambie después.

### Identidad durable

Cada invocation recibe:

- `sessionID`
- `agent`
- `messageID`
- `callID`

Los eventos terminales almacenan output/content/error de manera autocontenida.

### Permisos

El registry no inyecta genéricamente un `assertPermission`. Los trusted tools capturan `PermissionV2` y formulan su propia autorización antes del efecto.

## 7. Provider / AI stack

### `dev`

`@opencode-ai/llm` es schema-first y provider-neutral. Expone rutas/protocols para OpenAI, Anthropic, Gemini, Bedrock, Azure, OpenRouter, xAI, etc.

### `beta/v2`

La abstracción evoluciona a `@opencode-ai/ai`.

Además de LLM añade `Image` / `ImageClient` como primitive de primer nivel.

Principio explícito:

```text
provider quirks live in adapters, not calling code
```

El stream devuelve un `LLMEvent` común para:

- OpenAI Chat;
- OpenAI Responses;
- Anthropic Messages;
- Gemini;
- Bedrock Converse;
- OpenAI-compatible endpoints.

### Provider policy

V2 separa configuración de autorización:

```text
providers                  -> endpoint/options/model overrides
experimental.policies      -> whether provider.use is allowed
```

Evaluación:

- fallback allow;
- wildcard action/resource;
- last matching statement wins;
- user-global policy puede prevalecer sobre repo policy;
- managed organization policy está diseñado para tener autoridad final.

Esto reemplaza conceptualmente `enabled_providers` / `disabled_providers`.

## 8. Catalog, config y plugins

El decision record V2 documenta dos diseños y marca como seleccionado **Catalog Transforms**.

Cada plugin instala una transformación replayable sobre un `Catalog.Editor`.

Al cambiar:

- models.dev;
- auth;
- plugin activation;
- plugin disablement;
- config;
- policy;

el catalog se reconstruye reproduciendo transforms activas y aplicando policy al final.

Ventaja principal: cerrar el Scope de un plugin elimina automáticamente su transformación sin escribir lógica inversa manual.

## 9. Event architecture

El endpoint HTTP de eventos usa un único feed codificado por Server y una cola finita independiente por conexión.

```text
Core EventV2
    |
one global listener
    v
Server EventFeed
 filter + schema encode + JSON + SSE once
    |
    +--> queue client A
    +--> queue client B
    +--> queue client C
```

Cada subscriber tiene una dropping queue de 4096 eventos públicos.

Si un cliente desborda:

- sólo esa queue es expulsada;
- publicación global no se bloquea;
- otros clientes reciben el mismo evento en orden.

V2 documenta benchmarks donde eliminar encoding por cliente reduce drásticamente coste con 10/50 subscribers.

### Durable vs live-only

`sessions.log()` sirve eventos durable/replayable por aggregate sequence.

Los deltas live-only de texto/reasoning/tool input no forman parte de ese log. Así se separan claramente:

- state/recovery semantics;
- rendering incremental de UI.

## 10. Persistence

V2 usa SQLite/Drizzle pero el aspecto clave no es la tecnología sino la semántica:

- durable events;
- transactional projectors;
- durable inbox;
- instruction blobs/state;
- execution claims;
- projected messages/tool state;
- migrations desde V1.

PR #40723 añade migración V1 -> V2 con progreso resumible e import de legacy JSON credentials.

**INFERENCIA.** La persistencia V2 está diseñada para que UI, recovery y execution lean proyecciones reconstruibles, en vez de usar objetos runtime como fuente principal de verdad.

## 11. Backend / protocol / SDK

La especificación V2 declara que `Protocol` posee las operaciones HTTP y los transport errors. `Server` compone ese protocolo con Core.

Los clientes Promise/Effect son generados desde el `HttpApi` ensamblado.

Esto contrasta con superficies legacy donde handlers y SDK pueden evolucionar más estrechamente ligados.

### `dev` actual

`packages/server` existe, pero es más delgado y `packages/opencode` sigue siendo composition center.

### V2

El package graph convierte `server` en boundary real y `client` en consumidor del contrato protocol/schema.

## 12. UI / Desktop / TUI

V2 tiene `packages/tui` como dependencia del CLI.

La línea beta demuestra que no es sólo un prototype backend:

- #41626 publica desktop beta con CLI V2;
- #41627 construye beta desde V2;
- PRs posteriores corrigen TUI, desktop y app específicamente para V2.

Ejemplos encontrados:

- V2 TUI session/project open menu;
- location inheritance para nuevas sessions;
- `/cd` antes de crear una session;
- Desktop overlay para session removals;
- app evita legacy config endpoints al detectar V2.

## 13. Diferencias frente a `dev`

### V2 va más lejos en

- usar `packages/cli` como root runtime;
- durable admission/execution separation;
- claims/recovery formalizados;
- event replay contracts;
- provider policy;
- catalog transforms scoped;
- client generation desde HttpApi;
- `@opencode-ai/ai` incluyendo image models;
- multi-runtime Core incluyendo `workerd` en varias implementaciones.

### `dev` actual incorpora

- core/server/cli extraídos;
- conditional Node/Bun implementations;
- LLM provider-neutral schema-first;
- Hono/Node portabilidad histórica;
- SQLite abstractions y Effect layers.

Pero todavía mantiene:

- `packages/opencode` como entrypoint principal;
- SDK legacy;
- semánticas legacy coexistiendo con nuevos packages.

## Conclusión

**HECHO.** V2 es una arquitectura completa y en uso real mediante el canal beta, no una colección de experiments aislados.

**INFERENCIA fuerte.** Su objetivo central es transformar OpenCode de una aplicación Bun/monolítica con subsistemas internos a una plataforma de servicios explícitos donde Core posee dominio durable, Protocol posee contratos, Server posee transport, CLI/UI son composition/clients y providers/tools/plugins son extensiones scoped sobre contratos estables.
