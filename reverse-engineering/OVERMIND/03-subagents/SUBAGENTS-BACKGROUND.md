# Subagents and Background Work

**Status:** RECOMMENDED; Overmind subagents are currently deferred

## OpenCode pattern worth adopting

OpenCode implementa un subagent como una child Session real detrás de `TaskTool`:

- parent/child identity;
- agent target explícito;
- optional model override;
- derived permissions;
- depth limit;
- resume por child session ID;
- foreground/background execution;
- cancellation linkage;
- result reintroduced into parent.

La identidad separada es la lección principal.

## Overmind target

No crear `Subagent` como llamada recursiva al mismo `Agent._units`.

```text
Parent Session/Execution
      |
      +-- spawn request
              |
              v
         Child Session
         Child Execution
         Child Agent state
         own ContextCompiler view
         explicit ModelTarget
         explicit Tool/Plugin grants
         own budgets
              |
              v
         SubagentResult
              |
              v
 Parent receives bounded ProtocolUnit/observation
```

## Contract mínimo

```text
SubagentSpec
- profile/role
- task
- target_id
- tool grants
- context seed policy
- max model turns
- token/cost/time budget
- depth limit
```

`SubagentResult` debe contener bounded content + structured status/usage/artifact references, no copiar toda child history al parent context.

## Context isolation

El child puede recibir un **explicit context seed** producido por Core, no referencia mutable a parent `_units`. El parent decide qué resultado incorporar mediante su protocolo/context rules.

## Resume

Después del MVP, `child_session_id` permite continuar la misma child. Resume debe rehidratar canonical units y compiler state como cualquier Session.

## Background

Implementar solo después de:

- EventPort;
- Persistence;
- RunState;
- cancellation;
- child sessions.

Background job no es una Tool “que tarda”; es una execution con lifecycle independiente. El parent recibe un FACT cuando termina y una policy decide si/whether wake cognition.

## Cognitive economy

Alineado con el propio diseño de Overmind: polling de un background child no debe despertar el modelo. El runtime espera/detecta determinísticamente y solo produce cognition cuando el contract lo exige.
