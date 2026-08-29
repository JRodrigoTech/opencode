# Tool Registry

**Status:** VERIFIED-CODE

## Built-in initialization

El registry inicializa definitions para invalid, shell, read, glob, grep, edit, write, task, webfetch, todowrite, websearch, skill, apply_patch y, según flags/client, question, LSP, plan-exit y Code Mode `execute`.

## Custom local tools

Se escanean config directories con:

`{tool,tools}/*.{js,ts}`

Cada módulo compatible se adapta a `Tool.Def`. El boundary soporta Zod args y definiciones JSON Schema legacy. El registry no entrega directamente estos módulos al provider: los normaliza primero al contrato host.

## Plugin tools

Los hooks de plugin pueden exponer `p.tool`; se convierten al mismo `Tool.Def`. El wrapper aplica el mismo contexto de ejecución y truncation que las tools host.

## Model/provider gating

### Web search

Visible cuando provider es `opencode` / `opencode-go` o flags Exa/parallel lo habilitan.

### ApplyPatch vs Edit/Write

`usePatch` es verdadero para model IDs que contienen `gpt-` y no contienen `oss` ni `gpt-4`:

- `apply_patch` visible si `usePatch`;
- `edit` y `write` visibles si no `usePatch`.

El toolset puede cambiar por model ID aunque agent y permissions sean idénticos.

## Agent/session-aware resolution

`registry.tools()` recibe provider ID, model ID, agent y permissions de session. Después `SessionTools.resolve()` construye el Tool.Context y vuelve a aplicar la policy efectiva al solicitar `ctx.ask`. Por tanto hay dos planos distintos:

1. **capability visibility** — qué definición ve el modelo;
2. **authorization** — si una invocación concreta puede ejecutarse.

## Code Mode gating

Aunque `execute` exista por flag, se elimina si `describeCodeMode()` no puede construir un catálogo MCP visible tras permissions.

## Definition hook

Antes de devolver una definition se ejecuta:

`plugin.trigger("tool.definition", { toolID }, output)`

El plugin puede alterar description, parameters y/o JSON schema.

## Task description

La definición de `task` incluye dinámicamente agents cuyo mode no es primary y que no están denegados para el parent mediante `Permission.evaluate("task", agentName, parent.permission)`.

## Sources

- `packages/opencode/src/tool/registry.ts` — `9167cb3ea6bc5c8dd075f0f8271adbdec6074b12`
- `packages/opencode/src/session/tools.ts` — `0f401c7562fa07076afd539990ca12fa207ceee0`
