# Phased Implementation Plan

## Phase A — Freeze invariants before infrastructure

1. Completar/reforzar refactor de ContextCompiler sin cambios semánticos.
2. Serializers versionados para UserUnit/AssistantUnit/ProtocolUnit.
3. Tests de round-trip canónico.
4. Stable ID value types.

**Exit:** canonical cognitive state puede serializarse/reconstruirse sin provider messages.

## Phase B — EVENTS + Session shell

1. Implementar local EventPort FACT/SIGNAL según contract existente.
2. Session/Execution records in-memory primero.
3. RunState con one-active-execution invariant.
4. Emitir FACTs desde commit boundaries del Agent/Tool execution.

**Exit:** una ejecución puede observarse completamente sin parsear logs/callbacks ad hoc.

## Phase C — Persistence

1. SessionRepository.
2. required sink idempotente para durable FACTs.
3. rehydrate Session canonical units.
4. crash/restart tests.

**Exit:** restart -> resume conserva Context semantics y Tool protocol.

## Phase D — ToolExecutor + PermissionService

1. ToolExecutionContext.
2. CapabilityResolver.
3. Permission rules/approver port.
4. lifecycle events.
5. integrate existing Workspace tools without weakening confinement.

**Exit:** every side-effecting action is identifiable, cancellable, policy-checked and observable.

## Phase E — MutationJournal

1. wrap existing Workspace recovery.
2. before/after fingerprints.
3. rollback/reconciliation API.
4. execution linkage.

**Exit:** committed filesystem mutations have audit/recovery identity.

## Phase F — Foreground subagents

1. child Session + explicit spec.
2. depth=1.
3. context seed/result boundary.
4. independent model/tool budgets.
5. cancellation propagation.
6. resume child.

**Exit:** child work never aliases parent context or capabilities implicitly.

## Phase G — SERVICES/background

1. implement deferred Service lifecycle.
2. BackgroundJob service.
3. no-poll completion events.
4. explicit wake/cognition policy.

## Phase H — MCP + Skills

Implementar como Plugins sobre substrate, no como Core branches.

## Phase I — Runtime API / WebUI / ACP

Stabilize public API after operational semantics exist.

## Phase J — Additional model backends/routing

Solo según requirements reales. Mantener explicit targets.
