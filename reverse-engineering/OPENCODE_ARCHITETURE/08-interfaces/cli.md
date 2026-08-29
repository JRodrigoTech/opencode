# CLI / TUI Surface

**Status:** VERIFIED-CODE at package boundary

## Packages

Production contiene `packages/cli` con:

- `src/index.ts`
- `src/tui.ts`
- `src/commands/`
- `src/framework/`
- `src/services/`.

El package `opencode` también contiene command/session/server runtime usado por la superficie interactiva.

## Architectural position

CLI/TUI es una superficie de interacción sobre el runtime y no debe confundirse con el agent. Responsabilidades típicas:

- parseo de flags/commands;
- arranque/conexión al server/runtime;
- selección de modelo/agent/session;
- render de events/tool calls;
- permission/question interactions.

## Client-dependent tools

`ToolRegistry` habilita `question` por defecto solo cuando `RuntimeFlags.client` está en `app`, `cli`, `desktop`, salvo override `enableQuestionTool`.

`plan_exit` experimental se añade únicamente si plan mode experimental está habilitado y client es `cli`.

Por tanto, el client identity participa en capability resolution del agente.

## Source

- `packages/cli/src/**`
- `packages/opencode/src/tool/registry.ts`
- `packages/opencode/src/effect/runtime-flags.ts`
