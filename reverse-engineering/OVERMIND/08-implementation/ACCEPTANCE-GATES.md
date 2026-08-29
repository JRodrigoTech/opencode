# Acceptance Gates

Cada extracción desde OpenCode debería entrar solo con tests que demuestren que no rompe invariants de Overmind.

## Context gate

- provider messages siguen siendo compiled output.
- round-trip persistence reconstruye canonical units.
- required input no se trunca.
- ProtocolUnits completas siguen atómicas.
- Thinking no se reinserta automáticamente.
- summary source filtering permanece estable.

## Events gate

- FACT stable ID.
- deterministic sequence dentro de execution.
- required sink failure bloquea dependent commit.
- observer failure no reescribe FACT originario.
- SIGNAL drop no cambia semantic result.
- opaque reasoning nunca aparece en events públicos.

## Persistence gate

- restart/resume equivalence.
- idempotent replay by EventId.
- no Tool redispatch durante recovery.
- schema version mismatch falla explícitamente.
- secret/provider-private payloads no se almacenan accidentalmente.

## Tool/permission gate

- registry permanece frozen después de READY.
- capability visibility y invocation authorization testeadas por separado.
- deny falla cerrado.
- ask sin approver no auto-aprueba.
- cancellation llega a ToolExecutor.
- event metadata sigue bounded.

## Subagent gate

- child Session identity distinta.
- depth limit.
- explicit context seed.
- explicit target/tool grants/budgets.
- parent permissions no se heredan por accidente.
- cancellation propagation.
- child resume no duplica tool side effects.
- parent recibe bounded result, no child history completa.

## Background/services gate

- no polling LLM.
- no overlapping execution salvo contract explícito.
- deterministic start/stop.
- shutdown cancela o drena según policy.
- completion FACT durable antes de cualquier optional wake.

## MCP gate

- Core no importa MCP internals.
- remote tools pasan por PermissionService.
- resource size/content bounded.
- credentials no aparecen en events/context.
- reconnect no duplica registered tools.

## Interface gate

- CLI/WebUI/ACP usan RuntimeApiPort.
- disconnect no cancela semantic work salvo policy.
- event reconnect puede replay FACTs desde cursor.
- server auth no sustituye runtime permissions.

## Model gate

- backend específico no filtra wire types a Agent/Context.
- retry/recovery separation preservada.
- ModelCapabilities son explícitas.
- ToolCall partial/truncated sigue fail-closed.

## Definition of done

Una feature extraída de OpenCode no está terminada cuando “funciona en demo”; está terminada cuando añade capacidad sin ampliar autoridad accidental, sin romper canonical Context y con lifecycle/persistence/cancellation observables cuando corresponda.
