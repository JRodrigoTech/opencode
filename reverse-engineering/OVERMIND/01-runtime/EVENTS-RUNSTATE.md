# Runtime Events, RunState and Cancellation — extracción condicionada

**Status:** REFERENCE / ADAPT WHEN OVERMIND NEEDS IT

## FACT/SIGNAL sigue siendo buen diseño

Overmind ya tiene un contract deferred sólido:

- FACT = hecho committed con stable ID;
- SIGNAL = progreso efímero.

OpenCode confirma el valor de esta distinción, pero la integración inicial con OpenCode no exige implementar un EventPort global.

La Tool puede ejecutar una misión, consumir internamente el stream OpenCode y devolver un resultado final bounded.

## Cuándo promover EVENTS

Implementar EVENTS cuando una capability real de Overmind necesite un canal compartido, por ejemplo:

- WebUI streaming/reconnect;
- Memory ingestion de hechos committed;
- deterministic watchers/services;
- background work propio;
- auditabilidad cross-plugin.

## Delegation events futuros

Cuando EventPort exista, una external-agent capability puede mapear:

### FACT

```text
delegation.started
delegation.completed
delegation.failed
delegation.cancelled
```

### SIGNAL

```text
delegation.progress
delegation.permission_waiting
```

No es necesario traducir cada event interno de OpenCode a un event público Overmind.

## RunState

RunState genérico es útil si Overmind necesita executions cancellable/queryable más allá de una Tool call síncrona. Hasta entonces, mantener cancellation dentro del Tool/adaptor evita una nueva domain layer.

## Cancellation boundary

Si se usa ACP, OpenCode ya expone `cancel`. Si se usa subprocess one-shot, el adapter controla process lifetime. Esto resuelve la primera integración sin copiar `session/run-state.ts`.
