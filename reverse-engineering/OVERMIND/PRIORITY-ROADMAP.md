# Priority Roadmap

## P0 — Identity + durable facts

### 1. Stable IDs

Introducir tipos/values para:

- `SessionId`
- `ExecutionId`
- `EventId`
- `ToolCallId`
- opcional `ChildSessionId` como SessionId normal con parent link.

No sustituir `request_id` cognitivo; mapearlo a Execution metadata.

### 2. EventPort FACT/SIGNAL

Implementar primero el contract ya diseñado en `EVENTS_SERVICES_CONTRACT.md`.

FACT mínimos:

- `session.created`
- `request.started`
- `user.message.committed`
- `model.turn.completed`
- `tool.started`
- `tool.completed` / `tool.failed`
- `assistant.message.committed`
- `request.completed` / `request.failed`

SIGNAL mínimos:

- `assistant.stream.delta`
- `assistant.thinking.delta`
- `provider.activity`
- `tool.progress`

### 3. Persistence sink

Persistir FACTs y canonical ContextUnits suficientes para reanudar sin convertir provider messages en source of truth.

## P1 — Safe execution substrate

### 4. RunState

Un active foreground execution por session inicialmente. Cancellation token común para model, Tool y child jobs.

### 5. ToolExecutionContext

Añadir ID, cancellation, metadata/progress y permission ask al executor, no a cada Tool por separado.

### 6. PermissionService

Policy explícita allow/ask/deny con scopes. Diseñar sin necesidad de UI; `ask` puede ser port/handler y fallar cerrado si no existe interactive approver.

### 7. Subagent child sessions

Primero foreground + depth=1 + explicit agent spec/model target/tool grants/budget. Después resume.

### 8. MutationJournal

Generalizar el rollback de Workspace y registrar mutaciones con before/after identity y reconciliation status.

## P2 — Capability ecosystem

### 9. BackgroundJob/Service lifecycle

Después de event/persistence/cancel. No usar threads/sleep ad hoc por plugin.

### 10. MCP plugin

TOOLS + CONTEXT + opcional SERVICE. Todas las operaciones externas pasan por PermissionService y cancellation.

### 11. Skills

Representar Skills principalmente como ContextContributors versionados, con loader Tool solo si hay valor en lazy activation.

### 12. Runtime middleware tipado

Solo donde un caso real lo exija. Preferir observers de FACT/SIGNAL sobre hooks que mutan data.

## P3 — Remote surfaces

### 13. RuntimeApiPort

Operaciones de session, execute/cancel, event subscription, permission reply, state/query.

### 14. WebUI/HTTP

Adapter sobre RuntimeApiPort.

### 15. ACP

Adapter de sessions/content/tool/permission/usage sobre el mismo runtime.

## P4 — Optimizations/experiments

- Code Mode si MCP/tool-call round trips se convierten en bottleneck medido.
- smarter multi-model routing solo con evidencia; mantener target selection explícita como base.
- richer scheduler solo cuando una capability real requiera periodicidad.

## Dependencias que no deben invertirse

`WebUI -> RuntimeApi -> Session/Events`, nunca `WebUI -> Agent`.

`MCP -> public plugin ports`, nunca `Agent -> MCP internals`.

`Persistence <- FACTs`, nunca `Persistence -> provider payload construction`.

`Subagent -> ModelService`, nunca `Subagent -> backend`.
