# Persistence and Reversibility — separar estado de Overmind y estado delegado

## OpenCode delegation no requiere persistence general

Para el MVP, una Tool puede devolver:

```text
external_session_id
summary
artifact/change refs
```

El runtime actual puede conservar ese resultado como ProtocolUnit durante la sesión in-memory.

## Si Overmind implementa persistence

Debe hacerlo por sus propios objetivos: resume, Memory, WebUI, crash recovery, audit, etc.

Persistir entonces un reference record:

```text
DelegationRecord
- provider: opencode
- external_session_id
- task/result summary
- artifact refs
- status/timestamps
```

No importar toda la database/session history de OpenCode a Overmind.

## Reversibility

Las mutaciones hechas por `OpenCodeDelegateTool` ocurren **dentro del capability backend**. OpenCode posee sus mecanismos de coding state/revert y workspace semantics.

Overmind WorkspacePlugin ya posee recovery para sus propias write/delete Tools. No generalizarlo a `MutationJournal` solamente porque el agente delegado modifica archivos.

## Cuándo generalizar un journal

Solo si aparecen varias capabilities Overmind que requieren reconciliación uniforme de side effects y el Core necesita gobernar esa semántica.

Hasta entonces:

```text
Workspace mutations by Overmind -> Workspace recovery
Coding mutations by OpenCode    -> OpenCode/session responsibility
External APIs                    -> capability-specific reconciliation
```

## Uncertainty rule

Cancellation/transport failure después de una posible mutation no autoriza a reejecutar ciegamente la misión. Resume/reconcile con identity explícita.
