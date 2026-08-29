# Plugin System

**Status:** VERIFIED-CODE

## Plugin categories

OpenCode carga:

1. internal plugins importados directamente;
2. external configured plugins mediante `PluginLoader`;
3. legacy plugin exports compatibles.

Internal plugins de baseline incluyen integraciones de auth/provider como Codex, Copilot, GitLab, Poe, Cloudflare, Azure, DigitalOcean, Snowflake Cortex, xAI y Cerebras.

## Runtime state

`Plugin.Service` mantiene un array ordenado de `Hooks`. La carga externa es secuencial expresamente para mantener deterministic hook registration/execution order.

## Plugin input

Cada plugin recibe:

- OpenCode SDK client;
- project/worktree/directory;
- server URL;
- experimental workspace adapter registration;
- `Bun.$` cuando existe.

El SDK client puede apuntar al server real o llamar directamente al app fetch in-process.

## Trigger model

`Plugin.trigger(name,input,output)` recorre hooks en orden y espera cada hook antes del siguiente. El output mutable actúa como pipeline acumulativo.

Hooks observados en runtime incluyen:

- `tool.definition`
- `tool.execute.before`
- `tool.execute.after`
- `chat.message`
- `experimental.chat.messages.transform`
- `experimental.text.complete`
- `experimental.session.compacting`
- `shell.env`
- `command.execute.before`
- `experimental.chat.system.transform`
- event/config/dispose hooks.

## Event subscription

El plugin system escucha EventV2Bridge y reenvía eventos del directory actual a `hook.event`.

## Lifecycle cleanup

En finalizer se invoca `dispose` de cada hook. Errores de dispose/config se loguean y se ignoran para evitar derribar el runtime.

## Architectural interpretation

Plugins son interceptores de pipeline y providers de capabilities, no simples callbacks UI. Pueden afectar qué tool se define, qué argumentos se ejecutan y qué mensajes/contexto llegan al modelo.

## Source

- `packages/opencode/src/plugin/index.ts` — `6f05329a0833c0bb698572fba279e5bffc3bce49`
