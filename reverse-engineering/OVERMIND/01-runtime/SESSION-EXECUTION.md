# Session and Execution Runtime — referencia condicional

**Status:** ADAPT IF REQUIRED, not a prerequisite for OpenCode delegation

## Distinción importante

OpenCode demuestra el valor de una Session durable porque es un coding runtime stateful con múltiples interfaces. Esto **no obliga** a Overmind a construir primero un `SessionService` para poder invocar un coding agent.

Una Tool de delegación puede devolver un `external_session_id` OpenCode opaque y funcionar dentro del runtime actual.

## Cuándo necesita Overmind una Session propia

Introducir una Session domain de Overmind cuando aparezca una necesidad propia como:

- restart/resume durable de la conversación Overmind;
- WebUI + CLI simultáneos o multi-interface;
- Memory que necesite conversation identity estable;
- subagents nativos Overmind;
- background executions propios;
- cancel/query desde otro caller.

## Si se implementa

Mantener separadas al menos:

```text
OvermindSessionId
OvermindExecutionId
ExternalDelegationRef(provider="opencode", session_id="...")
```

No hacer equivalentes la Session de Overmind y la Session OpenCode.

## Context invariant

Una Session propia persiste canonical User/Assistant/Protocol units y compiler state suficiente. Nunca usa provider messages como canonical memory.

## Parent/child

Añadir parent/child identity solo cuando existan child executions Overmind. No es necesario para representar los subagents internos de OpenCode; esos permanecen detrás del adapter.
