# 02 — Entrypoints y runtime del agente

## Entrypoint CLI

**[CONFIRMADO]** `packages/opencode/src/index.ts` es el composition root del binario principal. Registra comandos para distintas superficies: TUI, ejecución non-interactive, attach, server headless, web, sesiones, providers, MCP, ACP y utilidades de operación/debug.

Esto significa que “OpenCode” no es un único modo de ejecución. El mismo runtime se puede consumir como:

- aplicación terminal interactiva;
- CLI non-interactive;
- servidor headless;
- backend embebido/sidecar de Desktop;
- endpoint para clientes externos;
- bridge ACP/MCP.

## Arranque de la TUI normal

Paths:

- `packages/opencode/src/cli/cmd/tui.ts`
- `packages/opencode/src/cli/tui/worker.ts`
- `packages/tui/src/app.tsx`

Flujo confirmado:

```text
proceso CLI
  -> resuelve project/cwd
  -> crea Worker backend
  -> crea cliente RPC
  -> usa transporte interno o servidor TCP
  -> arranca @opencode-ai/tui
  -> TUI consume backend por SDK
```

### Transporte interno

**[CONFIRMADO]** Sin flags de red, `TuiThreadCommand` no necesita abrir un puerto. Construye:

- URL lógica `http://opencode.internal`;
- un `fetch` que serializa la Request por RPC al Worker;
- un EventSource que recibe `global.event` por RPC.

En el Worker, `rpc.fetch` crea una `Request` y la entrega directamente a `Server.Default().app.fetch(request)`. El mismo árbol HTTP se ejecuta, pero sin socket TCP.

**[INFERENCIA]** Esta decisión mantiene una única semántica API entre modo embebido y modo remoto, evitando que la TUI tenga un camino privilegiado directo al runtime del agente.

### Transporte de red

**[CONFIRMADO]** Si el usuario solicita `--port`, `--hostname`, mDNS u opciones equivalentes, la TUI ordena al Worker arrancar `Server.listen()`. En ese caso usa una URL real y headers de `ServerAuth`.

## Server headless

`packages/opencode/src/cli/cmd/serve.ts` define `ServeCommand`.

**[CONFIRMADO]** El comando:

1. no crea una ambient `InstanceContext` al arrancar (`instance: false`);
2. resuelve opciones de red;
3. invoca `Server.listen`;
4. permanece activo mediante `Effect.never`.

El comentario del propio código establece que las instances se cargan por request usando `x-opencode-directory`.

## Desktop

`packages/desktop/src/main/index.ts` es el entrypoint del proceso main de Electron.

**[CONFIRMADO]** Desktop no incorpora el runtime del agente dentro del renderer. Supervisa un sidecar:

- camino V1: reserva un puerto loopback, genera password y ejecuta `spawnLocalServer`;
- camino V2, activado por `OPENCODE_SIDECAR_V2=1`: usa `startBackgroundCli`;
- publica al renderer la URL y credenciales cuando el backend queda disponible;
- administra lifecycle, shutdown, updater, deep links y sidecars WSL.

Esto vuelve a situar la API como boundary entre UI y runtime.

## Runtime de una sesión

El centro del agente se encuentra en:

- `packages/opencode/src/session/prompt.ts`
- `packages/opencode/src/session/processor.ts`
- `packages/opencode/src/session/run-state.ts`
- `packages/opencode/src/session/tools.ts`
- `packages/opencode/src/session/llm.ts`

### Flujo reconstruido de un turn

**[CONFIRMADO]** A alto nivel, el loop realiza las siguientes fases:

1. **Admisión de input.** Se materializa el mensaje/input del usuario dentro de la sesión.
2. **Serialización de ejecución.** `SessionRunState` evita que la misma sesión tenga loops incompatibles ejecutándose sin coordinación.
3. **Reconstrucción de contexto.** Se carga historial y se aplica la semántica de compaction/revert/continuation vigente.
4. **Trabajo sintético pendiente.** Antes de una llamada normal al modelo pueden procesarse subtasks, compaction u otras operaciones internas pendientes.
5. **Resolución de agente.** Se determina agente activo y sus restricciones/configuración.
6. **Resolución de modelo/provider.** Se obtiene provider/modelo y las opciones efectivas.
7. **Construcción del tool set.** Se filtran herramientas por agente, modelo, config, permisos, plugins/MCP y capacidad.
8. **Construcción del system/context.** Se combinan environment, instrucciones, prompt base/model-specific, instrucciones de proyecto, skills y contenido MCP cuando aplica.
9. **Invocación LLM.** `LLM` prepara la petición y abre el stream.
10. **Procesamiento incremental.** `SessionProcessor` consume text/reasoning/tool calls/usage/errors y actualiza mensajes/parts/steps.
11. **Tool execution.** Los tool calls son ejecutados mediante wrappers que aplican contexto, permisos y lifecycle; sus resultados vuelven a la conversación.
12. **Decisión de continuación.** Si el modelo produjo tools, hubo retry/compaction o queda trabajo interno, el loop continúa; si alcanzó estado terminal, finaliza.

## `SessionProcessor` como state reducer del stream

**[CONFIRMADO]** `SessionProcessor` no se limita a concatenar texto. Traduce eventos del stream a mutaciones de dominio, incluyendo:

- creación/actualización de parts;
- razonamiento y texto incremental;
- tool call pending/running/completed/error;
- step start/finish;
- usage/cost y metadatos;
- errores y retry semantics;
- finalización de assistant message.

**[INFERENCIA]** Puede entenderse como el reducer transaccional/operativo entre el protocolo de streaming del modelo y el modelo persistente de sesión.

## Control flow vs data flow

### Control flow

```text
SessionPrompt
  -> load state/history
  -> choose agent/model
  -> resolve tools
  -> build prompt
  -> LLM.stream
  -> SessionProcessor
       -> tool execution
       -> retry/error handling
       -> step completion
  -> loop / stop
```

### Data flow

```text
user input
  -> session message/input
  -> normalized model messages
  -> provider request
  -> provider stream
  -> assistant parts + tool parts
  -> events
  -> SQLite projections
  -> API events
  -> TUI/Desktop/SDK
```

## Agents y subagents

Paths relevantes:

- `packages/opencode/src/agent/agent.ts`
- `packages/opencode/src/agent/subagent-permissions.ts`
- `packages/opencode/src/tool/task.ts`
- campo `parentID` en el modelo de sesión.

**[CONFIRMADO]** El runtime representa tareas delegadas mediante sesiones relacionadas y dispone de reglas específicas para permisos de subagentes. El tool `task` es el punto visible de delegación desde el tool system.

**[INFERENCIA]** En lugar de introducir un runtime completamente distinto para subagentes, OpenCode reutiliza el modelo de sesión y el mismo pipeline de agente/tools/modelo, cambiando ownership, parentage y permisos.

## Compaction, continuation y retry

Paths:

- `session/compaction.ts`
- `session/overflow.ts`
- `session/retry.ts`
- `session/revert.ts`
- `session/summary.ts`

**[CONFIRMADO]** Estas capacidades forman parte del loop de sesión, no son postprocesos externos. El historial que llega al siguiente modelo puede ser reconstruido a partir de estado compactado/sintético y el loop puede reiniciarse después de condiciones recuperables.

## Invariante principal del runtime

**[INFERENCIA, fuertemente respaldada]** La unidad primaria de ejecución de OpenCode no es la request HTTP ni la llamada al LLM: es la **sesión durable**, sobre la que se serializan y reanudan múltiples pasos de modelo, herramientas y transformaciones de contexto.