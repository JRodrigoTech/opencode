# Runtime Dependency Graph

**Status:** DERIVED from VERIFIED-CODE

Este grafo enfatiza dependencias de comportamiento, no todos los imports.

```text
                         ┌──────────────┐
                         │   Clients    │
                         │ CLI/ACP/HTTP │
                         └──────┬───────┘
                                │
                         ┌──────▼───────┐
                         │   Session    │
                         │ Prompt/Loop  │
                         └──┬──┬──┬──┬──┘
                            │  │  │  │
             ┌──────────────┘  │  │  └───────────────┐
             ▼                 ▼  ▼                  ▼
        ┌─────────┐       ┌──────────┐          ┌──────────┐
        │ Agent   │       │ System   │          │ RunState │
        │ policy  │       │ Context  │          │ Runner   │
        └────┬────┘       └────┬─────┘          └──────────┘
             │                 │
             │             ┌───┴──────────────┐
             │             ▼                  ▼
             │        Instructions       Skills / MCP
             │
             ▼
        ┌──────────────┐
        │ SessionTools │◄──── ToolRegistry ◄──── Plugins/custom tools
        └──────┬───────┘
               │
               ├──────────► Permission
               │
               ▼
        ┌──────────────┐
        │     LLM      │
        └──────┬───────┘
               │
      ┌────────┴─────────┐
      ▼                  ▼
  AI SDK            Native LLM
 streamText       @opencode-ai/llm
      └────────┬─────────┘
               ▼
            LLMEvent
               │
               ▼
      ┌─────────────────┐
      │ SessionProcessor│
      └───┬────┬────┬───┘
          │    │    │
          │    │    └────► Snapshot/Patch
          │    └─────────► Session persistence
          └──────────────► Compaction / Retry / Status
```

## Effect layer graph

Los services se construyen mediante `Context.Service`, `Layer.effect` y `LayerNode.make`. Esto hace que las dependencias runtime estén representadas explícitamente en muchos módulos.

Ejemplos observados:

- `SessionProcessor.node` depende de Session, Config, Snapshot, Agent, LLM, Permission, Plugin, Summary, Status, Image, Event bridge y Database.
- `SystemPrompt.node` depende de Skill, MCP y LocationServiceMap.
- `Permission.node` depende del Event bridge.
- `SessionRunState.node` depende de BackgroundJob y SessionStatus.
- `Plugin.node` depende de Event bridge, Config y RuntimeFlags.

## Important cycles avoided by service boundaries

Aunque conceptualmente hay un ciclo “LLM → tool → session → LLM”, en implementación se rompe mediante interfaces:

- `SessionProcessor.Handle` expone actualización/completado de tool calls.
- `SessionTools.resolve` recibe un subconjunto de ese handle.
- `TaskPromptOps` proporciona operaciones de prompt al TaskTool sin importar el orquestador completo.
- `EffectBridge` adapta Effect a callbacks Promise del AI SDK/plugins.

Esto reduce dependencias circulares directas y conserva ownership.

## Data-plane vs control-plane

**Control plane interno:** Agent, Permission, Config, RuntimeFlags, Provider selection.  
**Data plane:** messages/parts, LLMEvents, tool args/results, snapshots/patches.  
**Extension plane:** Plugin, MCP, Skills, Code Mode.  
**External plane:** HTTP, ACP, CLI/client.

Esta separación es útil para auditar bugs: un error de “qué tool aparece” pertenece al control/capability plane; un tool result corrupto pertenece al data plane; una diferencia entre Claude/OpenAI puede aparecer en ProviderTransform o runtime LLM.
