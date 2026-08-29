# Extensibility — OpenCode como Plugin capability, MCP/Skills independientes

## OpenCodePlugin

El model Plugin actual de Overmind ya permite TOOLS + CONTEXT y stage/validate/freeze. Una integración básica con OpenCode puede ser simplemente un first-party Plugin que registra `agent.code`.

No necesita acceso al Agent object graph.

```text
OpenCodePlugin.register(context)
 -> context.tools.register(OpenCodeDelegateTool(...))
```

El adapter/proceso permanece dependency privada del Plugin.

## SERVICES — solo si hace falta long-lived ACP

Un one-shot subprocess puede vivir en la Tool.

Si se adopta un proceso `opencode acp` persistente con restart/readiness/shutdown, eso se convierte en un caso real para la categoría SERVICES ya prevista por Overmind. Entonces el service puede poseer el proceso/connection y la Tool consumir un port narrow.

No implementar SERVICES genérico antes del spike salvo que otra capability ya lo necesite.

## MCP

MCP es una capability distinta. Si Overmind lo necesita, encaja como Plugin propio:

- TOOLS remote;
- CONTEXT resources/instructions;
- SERVICES connection lifecycle cuando sea necesario.

No pasar por OpenCode MCP salvo que la intención sea específicamente que **el coding agent** use esa integración durante su misión.

## Skills

Mantener el diseño Overmind:

```text
SkillCatalog
 -> selected ContextContributor / Tool
 -> ContextBlock
 -> ContextCompiler
```

No copiar prompt mutation de otros runtimes.

## Boundary rule

Plugin capabilities pueden componerse:

```text
Overmind MemoryPlugin
Overmind RAGPlugin
Overmind OpenCodePlugin
Overmind MCPPlugin
```

pero ninguna necesita convertirse en Agent subclass ni acceder a internals de las demás.
