# Session and Execution Runtime

**Status:** RECOMMENDED

## Problema actual

`Agent` posee `_units` y `_request_id` in-memory. Es correcto para el Core actual, pero no basta para:

- restart/resume;
- WebUI/remote clients;
- child agents;
- background jobs;
- durable Memory ingestion;
- multi-session CLI;
- cancel/query desde otro caller.

## Lección OpenCode

OpenCode trata Session como dominio de identidad y persistence, no como variable local del CLI. Parent/child sessions, model/agent metadata, status, cost/tokens y revert forman parte del runtime.

## Adaptación propuesta

Separar tres identidades:

### Session

Conversación/lifetime durable.

```text
SessionRecord
- id
- parent_id?
- created_at / updated_at
- status
- metadata
- default agent/profile?
- default target?
```

### Execution

Un request activo o completado dentro de Session.

```text
ExecutionRecord
- id
- session_id
- cognitive_request_id
- state
- started/completed
- target_id
- input/output budget snapshot
- outcome/error
- usage/timing summary
```

### ModelTurn

Puede registrarse como FACT/event/record dentro de execution sin convertirse en top-level domain si no hace falta.

## State machine inicial

```text
created -> running -> completed
                   -> failed
                   -> cancelling -> cancelled
```

Un único foreground execution activo por Session es el default más seguro. Background child jobs son otra Session/Execution, no concurrencia arbitraria sobre el mismo Agent context.

## Reconstruction

SessionService rehidrata canonical ContextUnits y state necesario del compiler. Después entrega al Agent una conversación equivalente a la anterior. Agent no consulta SQL/JSON files directamente.

## Parent/child

`parent_id` debe ser first-class desde la primera versión del schema aunque subagents se implementen después. Evita migrar identidad cuando aparezca delegación.
