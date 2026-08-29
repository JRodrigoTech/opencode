# Overmind Current State

**Status:** VERIFIED-OVERMIND

## Qué es Overmind

Overmind se define como un **modular cognitive runtime built around a small, explicit AI Core**. El LLM es un reasoning engine, no el propietario del sistema. Esta diferencia debe gobernar cualquier importación de ideas desde OpenCode.

## Runtime implementado

El camino actual es:

```text
CLI / caller
    |
    v
 Runtime (READY boundary)
    |
    v
 Agent
    |
    +-> ContextCompiler -> CompiledContext
    |
    +-> ModelService -> GenerationExecutor -> ModelTarget -> OpenRouterBackend
    |
    +-> ToolPort / ToolRegistry -> Tool
```

`Agent` mantiene en memoria `_units` con `UserUnit | AssistantUnit | ProtocolUnit`. Cada request incrementa `request_id`, compila contexto y ejecuta hasta `max_model_turns`.

## Context

El Core implementa:

```text
ContextContributor
 -> ContextBlock[]
 -> ContextFrame
 -> ContextCompiler
 -> CompiledContext
 -> ModelService
```

Propiedades importantes:

- `messages[]` es transport output, no memoria canónica.
- Current user, first-user anchor, required blocks y ProtocolUnits completos tienen garantías explícitas.
- Token accounting es tokenizer-aware.
- Compaction histórica es bounded y puede activarse automáticamente, manualmente o vía tool `context.compact`.
- Sources ya compactadas permanecen en Agent state para auditabilidad pero dejan de reenviarse al provider.

## Model execution

`ModelService` usa mapping explícito de targets y exige `primary`. `GenerationExecutor` es el único cross-attempt loop.

Overmind ya separa correctamente:

- physical backend invocation;
- technical retry;
- normalized-response recovery/continuation;
- provider Thinking;
- authoritative Usage;
- continuation preflight.

Esta separación es superior a copiar un provider registry complejo antes de necesitarlo.

## Tools

`ToolRegistry`:

- exige canonical names `namespace.name`;
- valida schemas;
- reserva `context.compact` para Core;
- devuelve defensive schema copies;
- congela registration después de composition;
- ejecuta calls completos y produce `ToolExecutionResult` con observation/content/metadata bounded.

El Workspace plugin aporta actualmente cinco Workspace Tools; existen además tres Tools read-only de bootstrap.

## Plugins

`PluginComposer` hace stage -> validate -> commit -> freeze de TOOLS y CONTEXT. Los contracts target restringen plugins a public Core ports y categorías:

- TOOLS
- CONTEXT
- EVENTS (deferred)
- SERVICES (deferred)

El modelo de grants es explícito y no entrega el runtime completo al plugin.

## No implementado todavía

Según `AGENTIX/STATE.md`, permanecen deferred:

- persistent storage;
- Memory;
- RAG;
- Blackboard;
- EVENTS ejecutable;
- SERVICES ejecutable;
- Runtime API;
- subagents;
- MCP;
- scheduler/background attention;
- WebUI.

## Implicación para esta comparación

Las siguientes fases de Overmind ya no necesitan rediseñar su Core cognitivo para parecerse a OpenCode. Necesitan añadir un **operational shell** alrededor de ese Core: sessions, events, persistence, permissions, child execution y adapters externos.
