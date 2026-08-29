# Persistence, Mutation Journal and Reversibility

## Persistence

**RECOMMENDED:** introducir persistence como required EVENT sink + Session repository, no dentro de ContextCompiler.

### Qué persistir

- Session metadata/parent relationship.
- Execution lifecycle.
- canonical User/Assistant/Protocol units versionados.
- committed Thinking records si forman parte del transcript.
- model usage/timing/provider failure evidence.
- ToolCall/result metadata bounded.
- Context summary state/source IDs necesarios para resume exacto.
- stable FACTs.

### Qué no usar como autoridad

- raw provider messages como canonical state;
- provider-private reasoning continuation state como durable cognitive memory;
- transient SIGNAL deltas para reconstrucción semántica.

## Reversibility

Overmind Workspace ya tiene reversible write/delete recovery. OpenCode demuestra el valor de elevar esto a execution history mediante snapshot/patch/revert.

### MutationJournal propuesto

```text
MutationRecord
- mutation_id
- session_id / execution_id / tool_call_id
- capability
- resource identity
- operation
- before fingerprint/ref
- after fingerprint/ref
- reversible: bool
- rollback descriptor/ref
- state: committed|reverted|reconciliation_required
```

El journal no necesita ser Core cognitivo. Puede ser un runtime port usado por side-effecting capabilities.

## Por qué no copiar Git snapshots literalmente

Overmind tiene foco Windows y un Plugin Workspace propio. La abstraction debe permitir:

- filesystem backup/restore;
- diff/patch cuando aplique;
- Git-aware optimization opcional;
- external APIs no reversibles con reconciliation metadata.

## Exactly-once practical rule

Una Tool side effect committed no se repite automáticamente por model retry o persistence failure. Esto ya está alineado con Overmind: retry físico no redispatcha Tools committed.

Persistence/event required sinks deben fallar cerrado después de side effect y marcar reconciliation si no se puede registrar consistentemente.
