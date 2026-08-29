# Overmind Current State y frontera con OpenCode

**Status:** VERIFIED-OVERMIND + architectural interpretation

## Qué es Overmind hoy

Overmind se presenta como **a modular cognitive runtime built around a small, explicit AI Core**. El LLM es un reasoning engine dentro de un sistema mayor; Core posee contexto, model execution, Tool protocol y runtime boundaries.

El camino actual es:

```text
CLI / caller
    |
 Runtime READY boundary
    |
 Agent
    +-> ContextCompiler -> CompiledContext
    +-> ModelService -> GenerationExecutor -> ModelTarget -> OpenRouterBackend
    +-> ToolPort / frozen ToolRegistry -> Tool
```

`Agent` mantiene canonical `UserUnit | AssistantUnit | ProtocolUnit` en memoria y ejecuta bounded model turns.

## Context actual

`ContextContributor -> ContextBlock -> ContextFrame -> ContextCompiler -> CompiledContext` es una decisión fundacional.

- provider `messages[]` es output de compilación;
- Tool exchanges completas son ProtocolUnits;
- token accounting/compaction pertenecen a Context;
- provider Thinking no se convierte automáticamente en input futuro.

OpenCode no debe cambiar este ownership.

## Model execution actual

`ModelService -> GenerationExecutor -> ModelBackend` ya separa:

- target/model resolution explícito;
- physical invocation;
- technical retry;
- normalized response recovery/continuation;
- usage/timing;
- provider-specific backend behavior.

Un OpenCode agent delegado **no es un ModelBackend**. Es una capability de nivel superior que ejecuta una misión completa con su propio model/tool loop.

## Tools y Workspace

Overmind ya dispone de ToolRegistry frozen y una WorkspacePlugin con cinco operations bounded: list, search, read, write y delete, con recovery para mutaciones.

Estas Tools son útiles para acciones deterministas pequeñas. No implican que WorkspacePlugin deba evolucionar hasta convertirse en un IDE/coding agent.

Para tareas como comprender una codebase grande, planificar un refactor, editar múltiples archivos, ejecutar tests iterativamente o usar LSP/subagents, OpenCode es un mejor boundary especializado.

## Plugins

La implementación actual de Plugins soporta TOOLS y CONTEXT mediante stage -> validate -> commit -> freeze. EVENTS y SERVICES están todavía deferred.

Esto favorece un MVP muy pequeño para OpenCode: un Plugin puede aportar una Tool que invoque un proceso externo de forma scoped. No es necesario implementar primero todo SERVICES/EventBus/persistence.

Un proceso ACP persistente sí podría convertirse más adelante en un caso real que justificase SERVICES lifecycle.

## Deferred en Overmind

Siguen deferred, entre otros:

- Memory/RAG/Blackboard;
- persistent storage;
- executable EVENTS/SERVICES;
- Runtime API/WebUI;
- MCP;
- subagents nativos;
- scheduler/background attention;
- additional model backends/routing.

La existencia de equivalentes en OpenCode no altera automáticamente ese estado.

## Nueva implicación de este estudio

El valor inmediato de OpenCode para Overmind es doble:

1. **specialized capability**: usar OpenCode directamente para software engineering;
2. **reference implementation**: consultar sus soluciones maduras cuando una futura capability general de Overmind necesite sessions, permissions, cancellation, events o adapters.

No se recomienda construir un “operational shell estilo OpenCode” como objetivo independiente.
