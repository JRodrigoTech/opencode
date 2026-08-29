# Runtime Events, RunState and Cancellation

**Status:** RECOMMENDED; aligns with Overmind normative deferred EVENTS contract

## FACT vs SIGNAL debe mantenerse

Overmind ya define correctamente:

- **FACT**: hecho committed con stable event ID.
- **SIGNAL**: progreso efímero que puede perderse sin cambiar corrección.

OpenCode demuestra por qué esta distinción importa: streams del provider, tool states, permission requests y session updates necesitan orden observable, pero no todos son durable state.

## Event vocabulary propuesto

### FACT

```text
session.created
execution.started
user.unit.committed
context.compaction.committed
model.turn.completed
tool.call.committed
tool.completed
tool.failed
mutation.committed
assistant.thinking.committed
assistant.unit.committed
execution.completed
execution.failed
execution.cancelled
child.created
child.completed
```

### SIGNAL

```text
assistant.content.delta
assistant.thinking.delta
provider.activity
provider.retrying
tool.progress
permission.waiting
service.progress
```

## Ordering

Cada execution debe emitir `seq` monotónico o equivalente. Stable `event_id` + `execution_id` + sequence permiten:

- UI replay;
- idempotent persistence sink;
- reconnect;
- testing exact transitions;
- background result delivery.

## Required sinks

El contract de Overmind ya contempla required sinks. Úsalos para persistence de hechos que no pueden perderse. Un sink falla => no fingir commit.

No hacer durable cada stream delta: persistir el FACT final y, si observability lo requiere, telemetry separada.

## RunState

RunState debe poseer:

- execution active por session;
- cancellation token;
- child/background job references;
- state transitions;
- wait/query primitives.

No debe poseer ContextCompiler ni ModelBackend.

## Cancellation tree

```text
cancel parent execution
 -> cancel current model operation
 -> signal ToolExecutor
 -> cancel foreground child executions
 -> policy decides background child handling
```

Side effects ya committed no se reejecutan ni se “deshacen” automáticamente por cancellation; MutationJournal/Reconciliation define qué es reversible.
