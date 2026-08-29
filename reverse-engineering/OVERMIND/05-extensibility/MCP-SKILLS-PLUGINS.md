# MCP, Skills and Plugin Evolution

## MCP

**RECOMMENDED AS PLUGIN, P2.**

Mapeo natural a Overmind:

```text
McpPlugin
- SERVICES: client/process connection lifecycle
- TOOLS: remote MCP tools
- CONTEXT: resources/instructions selected through contributors
- EVENTS: connection/tool/resource facts and signals
```

Core no importa MCP. MCP Tool execution pasa por ToolExecutor + PermissionService + cancellation.

### Resources

No transformar automáticamente cada MCP resource en prompt text. Exponer resource metadata y lectura bounded; el resultado puede convertirse en ContextBlock/File artifact mediante contrato explícito.

### OAuth/secrets

Secrets son plugin-owned/config-owned y nunca eventos genéricos. Permission to use a remote tool sigue separada de possession of credentials.

## Skills

OpenCode Skills combinan discovery, system listing y una Skill tool. En Overmind pueden encajar más limpiamente:

```text
SkillCatalog
 -> SkillDescriptor
 -> selected Skill ContextContributor
 -> ContextBlock(authority/budget/source)
```

Opcionalmente una `skill.load` Tool puede hacer lazy activation. La activación debe producir state/event explícito, no modificar system prompt fuera del compiler.

Una Skill es conocimiento/instruction capability, no un Agent necesariamente.

## Plugins

Preservar la invariant de Overmind: Plugins contribuyen por public ports y son removibles sin cambiar Agent algorithm.

OpenCode inspira algunos lifecycle points, pero no copiar el hook surface completo.

Preferencia:

1. FACT/SIGNAL observers para observability/integration.
2. explicit capability ports para operaciones.
3. typed middleware solo para casos donde modificar el flujo sea requisito real.

## Services

El contrato deferred de Overmind ya define in-process lifecycle. MCP connection manager, GitHub watcher o indexer pueden ser primeros consumers. Su existencia no implica model wake-up.
