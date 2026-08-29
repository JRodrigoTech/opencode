# 01 — Arquitectura general y boundaries

## Tesis arquitectónica

**[CONFIRMADO]** OpenCode `dev` es un sistema local-first con arquitectura cliente/servidor, organizado como monorepo Bun/TypeScript. `packages/opencode` sigue actuando como composition root principal para CLI y una parte importante del runtime, mientras varias capacidades han sido extraídas a paquetes especializados (`core`, `llm`, `server`, `tui`, `schema`, `protocol`, SDK y aplicaciones).

**[INFERENCIA]** La forma vigente de `dev` es mejor entendida como una **arquitectura de transición con compatibilidad explícita**: el producto mantiene contratos V1 operativos mientras introduce servicios, eventos, schemas y rutas V2 en paralelo.

## Capas observadas

### 1. Superficies cliente

- `packages/tui`: cliente terminal basado en OpenTUI/Solid.
- `packages/desktop`: proceso Electron que supervisa un sidecar backend y presenta la aplicación gráfica.
- `packages/app` y SDK: superficie reutilizable para clientes HTTP/eventos.
- CLI non-interactive (`run`, comandos de administración, etc.) en `packages/opencode/src/cli`.

**[CONFIRMADO]** La TUI recibe un `url`, un `fetch` opcional y un `EventSource`, y monta un `SDKProvider`; no contiene el loop del agente. El backend permanece detrás de la API incluso cuando se ejecuta en el mismo host.

### 2. Boundary HTTP / transport

Paths de alta señal:

- `packages/opencode/src/server/server.ts`
- `packages/opencode/src/server/routes/instance/httpapi/server.ts`
- `packages/opencode/src/server/routes/instance/httpapi/groups/*`
- `packages/opencode/src/server/routes/instance/httpapi/handlers/*`
- `packages/opencode/src/server/routes/instance/httpapi/middleware/*`

El servidor compone rutas root/globales, rutas asociadas a una instance, SSE de eventos y WebSocket para PTY. Las rutas nuevas extraídas a `@opencode-ai/server` conviven con handlers todavía definidos en `packages/opencode`.

### 3. Application/runtime

El núcleo operativo que todavía reside en `packages/opencode` incluye, entre otros:

- `session/*`: orquestación del agente, mensajes, compaction, retry, revert, summary y run state.
- `agent/*`: definición y selección de agentes.
- `tool/*`: herramientas ejecutables y registry.
- `permission/*`: autorización de tools.
- `provider/*`: modelos, providers, autenticación y compatibilidad.
- `config/*`: configuración efectiva por instance.
- `project/*`, `worktree/*`, `control-plane/*`: contexto de proyecto/workspace.
- `mcp/*`, `acp/*`, `lsp/*`: integraciones/protocolos.
- `plugin/*`: extensibilidad.

### 4. Core e infraestructura compartida

`packages/core` concentra primitives reutilizables y servicios con Effect:

- runtime/layers;
- database SQLite/Drizzle;
- eventos V2;
- session projections y nuevos modelos;
- project/workspace;
- filesystem/PTY abstractions con implementaciones Bun/Node;
- system context y session runner.

**[CONFIRMADO]** `packages/core/package.json` usa package imports condicionales `#sqlite`, `#pty` y `#fff` para seleccionar implementaciones según runtime Bun o Node. Esto demuestra que el boundary de infraestructura pretende soportar más de un host de ejecución.

### 5. Boundary LLM

`packages/llm` expone una capa explícita de providers y protocolos. Entre sus exports aparecen:

- Anthropic Messages;
- Bedrock Converse;
- Gemini;
- OpenAI Chat;
- OpenAI-compatible Chat;
- OpenAI Responses.

Además expone providers para Anthropic, OpenAI, Azure, Google, Bedrock, OpenRouter, xAI, GitHub Copilot y otros.

**[CONFIRMADO]** `packages/opencode/src/session/llm.ts` sigue actuando como adapter/orchestrator de sesión sobre ese stack: toma modelo, mensajes, tools y opciones de agente y aplica transformaciones/hook middleware antes de iniciar el stream.

### 6. Schema y protocol

`@opencode-ai/schema` es un paquete de contratos/schema de bajo nivel basado en Effect. `@opencode-ai/protocol` depende de schema y Effect y proporciona contratos de protocolo desacoplados de las aplicaciones.

**[INFERENCIA]** Esta separación reduce dependencias circulares y permite que server, SDK, clientes y runtime compartan wire contracts sin importar directamente el monolito `packages/opencode`.

## El boundary `Instance`

Uno de los conceptos estructurales más importantes de `dev` es la **instance**.

Evidencias:

- `packages/opencode/src/project/instance-runtime.ts`
- `packages/opencode/src/project/instance-store.ts`
- `packages/opencode/src/project/instance-context.ts`
- `packages/opencode/src/effect/instance-*`
- middleware `server/routes/instance/httpapi/middleware/instance-context.ts`
- `ServeCommand` declara `instance: false` y documenta que las instances se cargan por request mediante `x-opencode-directory`.

**[CONFIRMADO]** El server headless no necesita fijar un proyecto global al arrancar: puede resolver el contexto de ejecución por request. La configuración efectiva, provider state y otros servicios se resuelven dentro de ese contexto.

**[INFERENCIA]** `Instance` es el principal boundary de tenancy local del runtime: encapsula un directorio/proyecto/workspace y su estado asociado, permitiendo que un mismo proceso atienda más de un contexto sin depender de un único `cwd` global.

## Ownership de estado

| Estado | Owner principal observado | Persistencia / lifetime |
|---|---|---|
| Sesión y mensajes | Session runtime + projectors | SQLite; compatibilidad legacy |
| Ejecución de un turn | `SessionRunState` / `SessionProcessor` | memoria + writes incrementales |
| Config efectiva | `Config` bajo instance | calculada/cached; múltiples fuentes |
| Providers/modelos | `Provider` bajo runtime/instance | config + auth + plugins |
| Tool set | `ToolRegistry` + `SessionTools` | calculado por ejecución |
| UI local | TUI/Desktop | cliente; stores propios |
| Eventos | buses legacy + `EventV2` | transient + durable según tipo |
| Database | `Database.Service` | global SQLite |

## Boundary legacy / nuevo

**[CONFIRMADO]** La composición del server monta simultáneamente servicios/session APIs V1 y V2. `SessionProjector` consume eventos V1 y eventos nuevos para mantener proyecciones SQLite. `event-v2-bridge.ts` existe específicamente para conectar el mundo legacy con `EventV2`.

**[INFERENCIA]** La estrategia de migración busca conservar la semántica observable para clientes mientras se sustituye gradualmente el modelo interno por eventos durables, servicios Effect y paquetes separados.

## Dependencias conceptuales

Flujo simplificado:

```text
TUI / Desktop / SDK / CLI
          |
          v
    HTTP / RPC / SSE / WS
          |
          v
 Server + instance routing
          |
          v
 Session application runtime
   |       |        |
 Agent   Tools    Provider/LLM
   |       |        |
   +-------+--------+
           |
           v
 Event bus / EventV2 / Projectors
           |
           v
      SQLite / storage
```

## Boundaries que parecen deliberados

**[CONFIRMADO]** Se observan interfaces claras en los siguientes puntos:

1. cliente ↔ server mediante SDK/HTTP/eventos;
2. request ↔ instance mediante routing/context;
3. session orchestration ↔ LLM mediante `LLM`/provider abstractions;
4. LLM ↔ provider wire protocol mediante `@opencode-ai/llm`;
5. session ↔ tools mediante registry + tool execution context;
6. writes/events ↔ read models mediante projectors;
7. host runtime ↔ infra mediante adapters Bun/Node.

**[INFERENCIA]** Estos seams son los mejores candidatos para reconstruir una arquitectura “ideal” de OpenCode: varios ya son packages independientes; otros todavía permanecen parcialmente incrustados en `packages/opencode`.