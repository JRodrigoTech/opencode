# 05 — Server, backend y protocolos

## Server como composition boundary

Los dos paths de mayor señal son:

- `packages/opencode/src/server/server.ts`
- `packages/opencode/src/server/routes/instance/httpapi/server.ts`

**[CONFIRMADO]** `Server.listen` usa el stack HTTP de Effect/Node y expone el mismo handler que puede consumirse in-process mediante `Server.Default().app.fetch`.

Esto permite dos topologías sin duplicar application semantics:

```text
TUI Worker --RPC--> app.fetch(Request)

cliente externo --HTTP--> Server.listen --> app.fetch(Request)
```

## Composición de rutas

El HTTP API está organizado en grupos y handlers separados. En `groups/*` aparecen, entre otros:

- global / instance / control / control-plane;
- project / workspace;
- session;
- provider;
- config;
- permission;
- question;
- MCP;
- PTY;
- files;
- sync;
- TUI;
- experimental.

**[CONFIRMADO]** `server.ts` mezcla rutas/handlers definidos todavía en `packages/opencode` con handlers extraídos a `@opencode-ai/server`.

**[INFERENCIA]** La extracción de server sigue el mismo patrón incremental que session y LLM: se conserva un composition root compatible mientras grupos completos se desplazan a packages con dependencias más estrechas.

## Middleware y request boundaries

El árbol de middleware incluye:

- authorization;
- compression;
- CORS/Vary;
- error mapping;
- fence;
- instance context;
- proxy;
- schema errors;
- workspace routing.

**[CONFIRMADO]** La selección de instance/workspace se realiza en la frontera HTTP y no debe ser reproducida individualmente por cada handler.

El comentario de `ServeCommand` confirma el uso de `x-opencode-directory` para cargar instances por request en modo headless.

## Lifecycle del server

`packages/opencode/src/server/server.ts` gestiona:

- bind hostname/port;
- mDNS cuando se habilita;
- listener lifecycle;
- cierre de conexiones HTTP;
- seguimiento/cierre de WebSockets;
- shutdown coordinado.

**[CONFIRMADO]** Un port `0` se usa para asignación dinámica cuando se solicita. Esto es especialmente útil en procesos embebidos/sidecars.

## Eventos hacia clientes

**[CONFIRMADO]** Existe una ruta/event transport para suscripción a eventos y la TUI puede recibir `GlobalEvent` tanto sobre la superficie server como mediante el bridge RPC interno del Worker.

**[INFERENCIA]** El event stream es el mecanismo primario de invalidación/sincronización de UI: los clientes consultan/read-models por API y reciben cambios incrementales para mantener estado local.

## PTY y WebSocket

El server define una ruta WebSocket específica para conexión a PTYs y dispone de `PtyTicket`/tracking de WebSockets.

**[CONFIRMADO]** La terminal interactiva es un subsistema de transporte distinto de la API request/response. Esto evita modelar streams bidireccionales de PTY como endpoints HTTP ordinarios.

## Session V1 y V2 en el mismo backend

La composición de layers del server incluye simultáneamente:

- Session legacy y servicios relacionados;
- `SessionProjector`;
- `EventV2Bridge`;
- `EventV2`;
- `SessionV2.node` y `SessionExecutionLocal.node`.

**[CONFIRMADO]** Ambas generaciones participan en el backend de `dev`.

**[INFERENCIA]** El server actúa como anti-corruption/composition layer durante la migración: permite que clientes y rutas legacy sigan funcionando mientras nuevas operaciones usan servicios V2.

## Protocolos de integración

### MCP

Paths de alta señal:

- `packages/opencode/src/mcp/index.ts`
- `mcp/auth.ts`
- `mcp/oauth-provider.ts`
- `mcp/oauth-callback.ts`
- `mcp/catalog.ts`
- rutas HTTP MCP y `cli/cmd/mcp.ts`.

**[CONFIRMADO]** MCP está integrado como fuente de tools/recursos externos y tiene lifecycle/auth propios, incluyendo OAuth.

**[INFERENCIA]** Internamente MCP queda normalizado hacia conceptos del runtime (principalmente tools y contexto) para que `SessionPrompt` no necesite tratar cada servidor externo como provider LLM.

### ACP

El árbol `packages/opencode/src/acp/` contiene agent, session, event, tool, permission, usage, content, directory y service, además del comando CLI ACP.

**[CONFIRMADO]** ACP funciona como un adapter de protocolo sobre conceptos internos de agente/sesión/tool, no como la implementación primaria de esos conceptos.

### LSP

`packages/opencode/src/lsp/` implementa cliente, server/language launch y diagnostics. El tool `lsp.ts` expone capacidades LSP al agente.

**[CONFIRMADO]** LSP es infraestructura auxiliar del tool runtime; el modelo no habla LSP directamente.

## Auth y exposición local

`ServeCommand` avisa si `OPENCODE_SERVER_PASSWORD` no está configurado. TUI, cuando usa un transporte de red, obtiene headers de `ServerAuth`; el Worker interno también puede inyectar autorización antes de llamar `app.fetch`.

**[CONFIRMADO]** La misma aplicación puede operar como backend estrictamente local/in-process o como servidor accesible por red, pero el segundo modo introduce una frontera de autenticación explícita.

## Boundary backend resumido

```text
HTTP / internal fetch
       |
       v
middleware: auth + routing + instance/workspace
       |
       v
API groups / handlers
       |
       +--> Session runtime
       +--> Config / Provider / Agent / Tools
       +--> MCP / PTY / Files / Project
       |
       v
Events + Projectors + Database
```

## Hallazgo principal

**[INFERENCIA]** El backend de `dev` no debería describirse como “un API encima del agente”. Es el **host de application services** que crea scopes de instance, compone implementaciones legacy/nuevas y expone transports uniformes a cualquier cliente. Por eso constituye uno de los boundaries arquitectónicos más estables del sistema.