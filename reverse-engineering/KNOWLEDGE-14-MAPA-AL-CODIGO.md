# 14 — Mapa al código: dónde mirar cuando una explicación ya no basta

Este archivo conecta preguntas de arquitectura con paths de alta señal en la baseline `dev` auditada.

## “¿Dónde empieza OpenCode?”

- `packages/opencode/src/index.ts` — composition root del binario/CLI principal.
- `packages/opencode/src/cli/cmd/tui.ts` — arranque TUI.
- `packages/opencode/src/cli/cmd/serve.ts` — server headless.
- `packages/desktop/src/main/index.ts` — Electron main.

## “¿Dónde está el loop del agente?”

- `packages/opencode/src/session/prompt.ts` — `SessionPrompt`, path de producto.
- `packages/opencode/src/session/processor.ts` — reducer del stream.
- `packages/opencode/src/session/run-state.ts` — runner ownership/cancelación.
- `packages/core/src/session/runner/` — arquitectura runner V2.

## “¿Cómo se monta el prompt/contexto?”

- `packages/opencode/src/session/system.ts`
- `packages/opencode/src/session/instruction.ts`
- `packages/opencode/src/session/compaction.ts`
- `packages/opencode/src/session/overflow.ts`
- `packages/core/src/system-context/`
- `packages/core/src/session/context-epoch.ts`
- `packages/core/src/session/history.ts`

## “¿Cómo funcionan agents/subagents?”

- `packages/opencode/src/agent/agent.ts`
- `packages/opencode/src/agent/subagent-permissions.ts`
- `packages/opencode/src/tool/task.ts`
- `packages/opencode/src/background/job.ts`

## “¿Cómo funcionan skills?”

- `packages/opencode/src/skill/index.ts`
- `packages/opencode/src/skill/discovery.ts`
- `packages/opencode/src/tool/skill.ts`
- `packages/opencode/src/session/system.ts`

## “¿Dónde viven las tools?”

- `packages/opencode/src/tool/tool.ts` — contrato.
- `packages/opencode/src/tool/registry.ts` — materialización/discovery.
- `packages/opencode/src/session/tools.ts` — tools de sesión.
- `packages/opencode/src/session/llm/request.ts` — filtro/proyección al modelo.
- `packages/opencode/src/tool/truncate.ts` — output grande.
- `packages/opencode/src/tool/json-schema.ts` — normalización de schemas.

### Tools sensibles

- `packages/opencode/src/tool/shell.ts`
- `packages/opencode/src/tool/apply_patch.ts`
- `packages/opencode/src/tool/edit.ts`
- `packages/opencode/src/tool/code-mode.ts`
- `packages/opencode/src/tool/external-directory.ts`

## “¿Dónde se decide un permiso?”

- `packages/opencode/src/permission/index.ts`
- `packages/core/src/v1/permission.ts`
- `packages/opencode/src/agent/subagent-permissions.ts`

## “¿Cómo habla con los modelos?”

### Path de producto

- `packages/opencode/src/session/llm.ts`
- `packages/opencode/src/session/llm/request.ts`
- `packages/opencode/src/session/llm/native-runtime.ts`
- `packages/opencode/src/provider/provider.ts`
- `packages/opencode/src/provider/transform.ts`

### Stack nativo

- `packages/llm/src/llm.ts`
- `packages/llm/src/provider.ts`
- `packages/llm/src/route/client.ts`
- `packages/llm/src/route/executor.ts`
- `packages/llm/src/cache-policy.ts`
- `packages/core/src/session/runner/model.ts`

## “¿Dónde está el estado durable?”

- `packages/core/src/event.ts`
- `packages/core/src/event/sql.ts`
- `packages/core/src/session/sql.ts`
- `packages/core/src/session/projector.ts`
- `packages/core/src/database/database.ts`
- `packages/opencode/src/event-v2-bridge.ts`

## “¿Dónde está el backend?”

- `packages/opencode/src/server/server.ts` — listener/application host.
- `packages/opencode/src/server/routes/instance/httpapi/server.ts` — composition de rutas/services.
- `packages/server/src/api.ts`
- `packages/server/src/handlers.ts`
- `packages/server/src/handlers/event.ts`
- `packages/protocol/src/api.ts`

## “¿Dónde está MCP?”

- `packages/opencode/src/mcp/index.ts`
- `packages/opencode/src/mcp/catalog.ts`
- `packages/opencode/src/mcp/oauth-provider.ts`
- `packages/opencode/src/mcp/auth.ts`
- `packages/opencode/src/server/routes/mcp.ts`

## “¿Dónde está ACP?”

- `packages/opencode/src/acp/agent.ts`
- `packages/opencode/src/acp/service.ts`
- `packages/opencode/src/acp/event.ts`
- `packages/opencode/src/acp/content.ts`
- `packages/opencode/src/acp/permission.ts`

## “¿Dónde está el grafo Effect?”

- `packages/opencode/src/effect/app-runtime.ts`
- `packages/core/src/effect/app-node.ts`
- `packages/core/src/effect/layer-node.ts`
- `packages/core/src/location-services.ts`

## Regla de investigación

Cuando un package nuevo parezca “la implementación definitiva”, busca antes **quién lo consume en el composition root**. Ese paso evita la mayoría de falsos positivos durante una migración arquitectónica.