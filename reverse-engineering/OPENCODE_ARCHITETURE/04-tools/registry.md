# Tool Registry

**Status:** VERIFIED-CODE

## Built-in initialization

El registry inicializa definitions para:

- invalid
- shell
- read
- glob
- grep
- edit
- write
- task
- webfetch (`fetch` internamente en el aggregate)
- todowrite
- websearch
- skill
- apply_patch
- question (según client/flag)
- LSP (experimental)
- plan-exit (experimental plan mode en CLI)
- execute / Code Mode (experimental y solo cuando hay catálogo útil)

## Custom local tools

Se escanean config directories con glob:

`{tool,tools}/*.{js,ts}`

Cada módulo exportado compatible se convierte en `Tool.Def`. El boundary soporta tanto Zod args como legacy JSON Schema-like definitions.

## Plugin tools

Cada plugin hook puede exponer `p.tool`; el registry los convierte al mismo `Tool.Def` y aplica truncation de output para compatibilidad operativa.

## Model/provider gating

### Web search

Visible cuando provider es `opencode` / `opencode-go` o flags Exa/parallel lo habilitan.

### ApplyPatch vs Edit/Write

`usePatch` es verdadero para model IDs que contienen `gpt-` y no contienen `oss` ni `gpt-4`.

- `apply_patch` visible si `usePatch`.
- `edit` y `write` visibles si no `usePatch`.

Esto significa que el toolset cambia por model ID aunque agent/permissions sean idénticos.

## Code Mode gating

Aunque `execute` exista por flag, se elimina si `describeCodeMode()` no produce una descripción — lo que ocurre, por ejemplo, si no hay MCP tools visibles tras permissions.

## Definition hook

Antes de devolver una definition al caller, se ejecuta:

`plugin.trigger("tool.definition", { toolID }, output)`

El plugin puede cambiar description, parameters y/o JSON schema.

## Task description

El registry concatena a `task` una lista dinámica de agents invocables desde el current agent.

## Sources

- `packages/opencode/src/tool/registry.ts` — baseline production
