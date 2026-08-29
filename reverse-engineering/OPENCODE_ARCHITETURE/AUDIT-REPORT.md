# Second-Pass Audit Report

**Audit date:** 2026-08-29  
**OpenCode source baseline:** `production@df35e842f59bc115bb7c0479a8e11f017d443f2c`  
**Original documentation commit:** `f540bc575606064912a589fb713fcbc8ac726ea4`

## Resultado

Se realizó una segunda pasada estática sobre la documentación de `OPENCODE_ARCHITETURE`, volviendo a cruzar las afirmaciones de comportamiento contra la baseline congelada de `production` y no contra el nombre móvil de la branch.

**Conclusión:** no se encontraron contradicciones arquitectónicas mayores en el modelo documentado. El loop, la state machine del processor, resolución de tools, permisos, model/provider seam, compaction, sessions, plugins/MCP y superficies externas siguen alineados con el código inspeccionado.

Sí se encontraron defectos menores de auditabilidad y omisiones relevantes, corregidos en esta pasada:

1. `SOURCE-INDEX.md` no fijaba el blob SHA de `tool/registry.ts`; queda fijado en `9167cb3ea6bc5c8dd075f0f8271adbdec6074b12`.
2. `04-tools/registry.md` pasa a incluir el SHA exacto anterior.
3. `05-llm/native-runtime.md` pasa a fijar `native-runtime.ts@bac385c59137ced710073051ed6388bc376e39ab` y aclara el gating exacto.
4. `03-agents/subagents.md` se amplía con evidencia de `TaskTool`: `task_id` resume, depth limit, child permissions, model inheritance, background jobs experimentales e inyección del resultado en el parent.
5. Se documenta explícitamente el límite de esta auditoría: es una auditoría de código estática; equivalencia temporal de streams, comportamiento de providers reales y race conditions requieren pruebas dinámicas.

## Matriz de revalidación

| Área documental | Fuentes revalidadas | Resultado |
|---|---|---|
| `00-architecture` | package boundaries, Effect LayerNodes, host/runtime imports | consistente |
| `01-agent-runtime` | `session/prompt.ts`, `processor.ts`, `run-state.ts`, `retry.ts` | consistente |
| `02-context-engine` | `system.ts`, `instruction.ts`, `message-v2.ts`, request preparation | consistente |
| `03-agents` | `agent.ts`, `tool/task.ts`, subagent permissions | consistente; ampliado |
| `04-tools` | `tool/registry.ts`, `session/tools.ts`, Permission | consistente; trazabilidad corregida |
| `05-llm` | `session/llm.ts`, `llm/native-runtime.ts`, `packages/llm` | consistente; gating precisado |
| `06-memory-state` | Session SQL/model, compaction, snapshot/revert | consistente |
| `07-extensibility` | MCP, Plugin, Skill, Code Mode | consistente |
| `08-interfaces` | ACP, server package, generated client contract, CLI surface | consistente |

## Hotspots reabiertos durante la auditoría

### Agent/model selection

`createUserMessage` confirma la precedencia:

`input.model ?? agent.model ?? currentModel(session)`.

La variant del agent solo se adopta automáticamente cuando el model seleccionado coincide con el model configurado por el agent y esa variant existe para ese model.

### Tool policy

`ToolRegistry.tools()` no es un inventario pasivo. El toolset final depende de provider, model ID, agent, session permissions y runtime flags. `SessionTools.resolve()` vuelve a combinar agent + session permissions en `ctx.ask()`; visibilidad y autorización de una invocación concreta son decisiones distintas.

### Subagents

`TaskTool` demuestra que un subagent es una child session real. Puede reusar una child existente mediante `task_id`, tiene límite configurable de profundidad, recibe permisos derivados y puede ejecutarse mediante `BackgroundJob` cuando el flag experimental correspondiente está activo.

### LLM runtime seam

El runtime nativo solo es elegible para `openai`, `anthropic` u `opencode*`, con packages AI SDK OpenAI/OpenAI-compatible/Anthropic y auth compatible. Si no cumple, `session/llm.ts` hace fallback explícito a AI SDK. En ambos casos el processor recibe `LLMEvent`.

## Qué NO valida esta auditoría

La inspección estática no prueba por sí sola:

- equivalencia byte/event-order entre AI SDK y native runtime frente a providers reales;
- timing y races de cancelación bajo carga;
- comportamiento de MCP servers externos defectuosos;
- compatibilidad efectiva de todas las combinaciones provider/model;
- semántica de red del server bajo despliegues concretos.

Estos puntos deben tratarse como **dynamic validation**, no como defectos de la documentación estática.

## Fuentes críticas

- `packages/opencode/src/session/prompt.ts` — `0f85d44f209ba792065aeb951f0bd2e12b59fae8`
- `packages/opencode/src/session/processor.ts` — `20aa8a8404d8e5f50b0aafee5034ed6f1fa44382`
- `packages/opencode/src/session/llm.ts` — `a99f8acff20c5d64d0b6cb90df480218bb1daddc`
- `packages/opencode/src/session/llm/native-runtime.ts` — `bac385c59137ced710073051ed6388bc376e39ab`
- `packages/opencode/src/session/tools.ts` — `0f401c7562fa07076afd539990ca12fa207ceee0`
- `packages/opencode/src/tool/registry.ts` — `9167cb3ea6bc5c8dd075f0f8271adbdec6074b12`
- `packages/opencode/src/tool/task.ts` — `d8ca640cfba9a52d97e5180fda0ffa719910592b`
- `packages/opencode/src/permission/index.ts` — `2e27ff2424dbb000ea9ed7f73471769716ba40a1`
- `packages/opencode/src/agent/agent.ts` — `536a642fe49fb5211e66c2e2ad689856a03254c0`
