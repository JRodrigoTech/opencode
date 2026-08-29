# Evolución de branches relacionadas con tools, permissions y Code Mode

## Cómo leer este inventario

El repositorio conserva numerosas topic branches históricas. Muchas no fueron borradas al integrarse o abandonarse y posteriormente quedaron cientos/miles de commits por detrás de `dev`.

Por ello se distinguen tres niveles:

- **Evidencia aislable:** el diff o archivos de la branch permiten atribuir un cambio concreto.
- **Línea evolutiva:** varias branches describen el mismo problema, y el concepto puede rastrearse hasta `dev`, aunque el diff agregado esté contaminado.
- **UI/ergonomía:** branch relevante para presentación, pero no prueba un cambio en el runtime de seguridad.

## Familia 1 — Tool registry, catálogo y naming

Branches observadas:

- `kit/effectify-tool-registry`
- `effect-dynamic-tools`
- `deferred-tools`
- `random-tool-catalog`
- `fix-stale-tools`
- `fix-tool-ordering`
- `canonical-tool-names`
- `mcp-tool-names`
- `renamed-tool-execution`
- `tools-menu`
- `pinned-tools`

### Evidencia

`kit/effectify-tool-registry` es la branch más aislable: concentra su cambio principal en `packages/opencode/src/tool/registry.ts` y conecta el registry con services/instances Effect.

`dev` mantiene `ToolRegistry.Service`, por lo que la separación del registry como servicio sobrevivió.

`canonical-tool-names` está muy divergida, pero el baseline actual contiene repair de nombres en `session/llm.ts`: si una tool call llega con casing incorrecto y existe la versión lower-case, la corrige; si no, usa `invalid`.

### Reconstrucción

```text
mutable/static catalog
      |
      +--> normalize names/order
      +--> dynamic/plugin/MCP tools
      +--> deferred/contextual materialization
      `--> Effect ToolRegistry Service (dev)
```

## Familia 2 — Schema projection y compatibilidad

Branches observadas:

- `investigate-tool-schemas`
- `cpu-tool-schema`
- `tools-additional-properties` (familia histórica detectada en búsquedas temáticas)
- `kit/mcp-tolerate-bad-output-schemas`
- `kimi-mfjs-schema`
- `native-provider-schema`
- branches `tool/schema*` de generaciones históricas

### Estado final en `dev`

`packages/opencode/src/tool/json-schema.ts` normaliza Effect Schema a JSON Schema interoperable y `session/llm/request.ts` aplica ajustes provider-specific como `strict: false`.

**Inferencia:** la evolución muestra que “schema válido internamente” y “schema aceptado por todos los providers/MCP peers” son problemas diferentes; de ahí la capa de proyección explícita.

## Familia 3 — Tool input recovery, invalid calls y provider repair

Branches observadas:

- `tool-input-recovery`
- `tool-input-repair`
- `raw-tool-recovery`
- `null-tool-inputs`
- `kit/tool-effect-invalid`
- `duplicate-tool-repro`
- `anthropic-tool-order`
- `anthropic-tool-finish`
- `nxl/fix-anthropic-provider-tools`

### Estado final

`session/llm.ts` contiene `experimental_repairToolCall`: corrige nombres case-insensitive cuando puede y enruta calls irreparables a `invalid` incluyendo nombre/error originales.

**Inferencia:** la política final favorece convertir errores de protocolo/model output en una tool-result explicable al agente, en vez de romper siempre el turno completo.

## Familia 4 — Lifecycle, pending state, interruption y settlement

Branches observadas:

- `tool-declined`
- `tool-result-guard`
- `interrupt-tool-hang`
- `interrupt-tool-repro`
- `kit/effect-native-tool-interrupt`
- `fix/session-tool-bugs`
- `fix/bash-tool-settlement`
- `bash-tool-crash`
- `pending-tool-details`
- `pending-tool-error`
- `kit/tui-pending-tool-spinner`

### Estado final

`session/processor.ts` y `ToolPart` mantienen `pending -> running -> completed|error`; abort/cancel se propaga mediante Effect/AbortSignal. `message-v2.ts` evita reinterpretar interrupciones huérfanas como trabajo pendiente durante replay.

### UI vs runtime

`pending-tool-details`, `pending-tool-error` y `kit/tui-pending-tool-spinner` son evidencia de que el estado se presenta al usuario, pero la máquina de estados autoritativa está en session/runtime.

## Familia 5 — Tool results, output, truncation y memoria

Branches observadas:

- `tool-output`
- `tool-output-pin`
- `bound-tool-output`
- `fix-tool-outputs`
- `tool-result-api`
- `tool-result-guard`
- `issue-13770-tool-output-docs`
- `hide-empty-output`
- `remove-output-paths`
- `perf/tool-memory`
- `shell-output-preview`
- `shell-progress-tail`
- `shell-tail-output`
- `shell-output-flush`
- `shell-interrupt-output`

### Estado final

`tool/truncate.ts` implementa límites de líneas/bytes, spill a fichero y cleanup; shell dispone además de truncado incremental. `message-v2.ts` reduce resultados históricos para el context window.

**Hecho:** son mecanismos distintos: preservar output completo operativamente y reducir lo que vuelve al modelo.

## Familia 6 — Permission rework y resolución

Branches observadas:

- `permission-rework`
- `effect-test-permission-next`
- `kit/config-permission-effect`
- `kit/retrofit-permission-main`
- `permission-reply`
- `permission-alias`
- `agent-permission-order`
- `nxl/fix-permission-specificity`
- `fix/v2-headless-permission-policy`
- `opencode/permission-server-fallback`
- `auto-accept-permissions`
- `feat/auto-accept-permissions`
- `subagent-permission-inheritance`

### Generación `permission-rework`

**Evidencia aislable:** contiene `permission/next` y el modelo que converge en requests/rulesets/`allow|deny|ask`.

### Orden y specificity

`agent-permission-order` evidencia que el orden del ruleset fue un problema explícito.

`nxl/fix-permission-specificity` implementa un algoritmo alternativo de ranking por wildcards/literal length. Este algoritmo no está en `dev`: el baseline usa la **última regla coincidente**.

### Approval/reply

`permission-reply` y ramas UX evolucionan el protocolo de responder solicitudes. El backend actual sigue siendo autoritativo mediante `Permission.Service.ask()/reply()` y `Deferred`.

### Headless/server

`fix/v2-headless-permission-policy` y `opencode/permission-server-fallback` sugieren adaptación del mismo policy engine a clientes sin modal interactivo o con backend remoto.

**Inferencia:** el diseño separó deliberadamente policy/decision del rendering de approval para poder operar TUI, app, headless y transports distintos.

## Familia 7 — Permission UI

Branches observadas:

- `permission-highlight`
- `permission-modal-keyboard`
- `permission-panel`
- `kit/permission-flow-ux`

Estas branches se consideran principalmente presentation/interaction. No se usan como prueba de cambio en precedencia o authority salvo cambios backend demostrables.

## Familia 8 — Subagent authorization

Branches observadas:

- `subagent-permission-inheritance`
- `adjust-subagent-tool-description`
- `agent-permission-order`

`dev` contiene `deriveSubagentSessionPermission()` con herencia selectiva: parent denies + `external_directory`; capacidades propias del child; `task`/`todowrite` denied por defecto si no se declaran.

**Hecho final más fuerte que el diff histórico:** la política está codificada y comentada explícitamente en `packages/opencode/src/agent/subagent-permissions.ts`.

## Familia 9 — Shell/Bash

Branches observadas:

- `bash-tool-crash`
- `fix/bash-tool-settlement`
- `shell-tool-fallback`
- `shell-tool-policy`
- `shell-tool-test`
- `shell-exit-flake`
- `shell-exit-grace`
- `shell-exit-order`
- `shell-exit-status`
- `shell-spawn-ready`
- `shell-interrupt-output`
- `shell-output-flush`
- `shell-output-preview`
- `shell-progress-tail`
- `shell-tail-output`
- `shell-background-notice`
- `wsl-shell-env`
- `v2-shell-utf8`

### Líneas funcionales

1. **Policy/scanning:** `shell-tool-policy`, `shell-tool-fallback`.
2. **Process lifecycle:** `shell-exit-*`, `shell-spawn-ready`, settlement/crash.
3. **Output:** preview/tail/flush/interrupt.
4. **Portabilidad:** WSL/env/UTF-8.

`dev` integra las cuatro preocupaciones en `tool/shell.ts`.

## Familia 10 — Patch/edit

Branches observadas:

- `apply-patch`
- `patch-empty-hang`
- `patch-error-style`
- `patch-input-guidance`
- `patch-symlink-policy`
- `kit/collapse-patch`
- `optimize-apply-patch`

### Generación `apply-patch`

Es evidencia aislable y fuerte: añade `apply_patch`, elimina `patch`, cambia registry y reemplaza tests.

### Hardening posterior

Las branches restantes cubren inputs degenerados, symlink/path policy, errores, guidance y optimización. El `dev` final precomputa cambios, valida paths, pide `edit` y sólo entonces muta.

## Familia 11 — Code Mode

Branches observadas:

- `code-mode-boundary`
- `code-mode-opencode`
- `execute-code-mode-v2`

`execute-code-mode-v2` contiene la extracción/desarrollo del package `codemode` y en su punta el commit `c590e276398bfd3fbadbf2113144e0bece9bfaa8` (`feat(core): add grouped and deferred tool registration (#35232)`).

`dev` conserva el resultado arquitectónico: `execute` usa un intérprete confinado y un catálogo MCP filtrado, con autorización por child call.

## Familia 12 — MCP/tool catalog crossover

Branches observadas:

- `mcp-search-tool`
- `mcp-tool-names`
- `kit/mcp-tolerate-bad-output-schemas`
- `code-mode-*`

Esta familia cruza AGENT 08 (MCP) y AGENT 05. Aquí sólo importa la frontera tool-runtime: MCP tools se transforman a la misma superficie autorizable y, en Code Mode, a capabilities namespaceadas.

## Familia 13 — UI / rendering de tool rows

Branches observadas incluyen:

- `tool-row-typography`
- `tool-flat-draft`
- `pending-tool-details`
- `tools-menu`
- `new-toolbar-layout`
- `tooltip-delay`
- `context-tooltip-v2`

Se catalogan para completar el inventario, pero no son evidence de cambios del executor salvo modificación backend adicional.

## Mapa evolutivo resumido

```text
             +---------------- tool schema/projection ----------------+
             |                                                        |
registry ----+--> dynamic/deferred tools --> Effect ToolRegistry -----+--> LLM
   |                                                                  |
   +--> canonical names / input repair -------------------------------+

LLM tool call
   |
   v
ToolPart lifecycle ---- interruption/settlement ---- replay guards
   |
   v
Permission engine ---- rework ---- ordering/specificity experiments
   |                      |
   |                      `--> UI/headless/server reply surfaces
   |
   +--> subagent inherited restrictions
   +--> external_directory
   |
   v
side-effect tool
   +--> shell process boundary
   +--> apply_patch/edit filesystem boundary
   `--> Code Mode child capability boundary

result --> truncation/output spill --> historical context reduction
```

## Qué sobrevivió claramente en `dev`

- Effect-based `ToolRegistry.Service`.
- Typed Tool contract + schema projection.
- Explicit ToolPart lifecycle.
- Tool-call repair/`invalid` fallback.
- `allow|ask|deny` backend permission service.
- Ordered last-match permission precedence.
- `Deferred` approval waiting.
- Selective subagent permission inheritance.
- External-directory checks.
- Shell parser/policy + robust process settlement + incremental output.
- `apply_patch` as replacement of legacy `patch`.
- Code Mode as confined capability runtime over MCP tools.
- Centralized output truncation plus history reduction.

## Ideas experimentales o no adoptadas literalmente

- **Specificity-based permission ranking:** visible en `nxl/fix-permission-specificity`; no es el algoritmo vigente.
- **Branches de auto-accept:** no sustituyeron la policy backend general por un bypass universal.
- **Portable shell scanner como única ruta:** el changeset de `shell-tool-policy` lo describe como opt-in y mantiene la ruta tree-sitter por defecto en aquella evolución.

## Limitación metodológica

No puede inferirse una secuencia temporal exacta únicamente por el estado actual de todas las topic branches, porque muchas acumularon integración/rebases posteriores. Para branches altamente divergentes, este documento afirma sólo:

- existencia del experimento;
- objetivo inferible de su nombre/changeset/archivos temáticos;
- relación con mecanismos que sí son verificables hoy en `dev`.

Cuando el diff es aislable (`apply-patch`, `kit/effectify-tool-registry`, `nxl/fix-permission-specificity`), se hacen afirmaciones más fuertes.