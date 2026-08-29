# Capability Resolution and Runtime Permissions

## Current Overmind

**VERIFIED-OVERMIND:** ToolRegistry es frozen y Plugin construction grants son explícitos. Workspace limita filesystem scope. No existe todavía un general runtime permission broker.

## OpenCode lesson

OpenCode separa dos preguntas:

1. qué tool/capability ve el modelo;
2. si una invocación concreta está autorizada.

Esta separación es esencial cuando lleguen shell, MCP, browser, GitHub, subagents y external services.

## Diseño recomendado

### ToolRegistry

Mantenerlo casi como está: registro estático y frozen.

### CapabilityResolver

Nuevo pure/policy component por execution:

```text
registered tools
+ agent/profile grants
+ target/model capabilities
+ session policy
+ interaction surface capabilities
 -> visible ToolSchemas
```

No muta registry.

### PermissionService

Evalúa la acción concreta:

```text
permission key + resource pattern + execution context
 -> ALLOW | ASK | DENY
```

Ejemplos:

- `workspace.read:path`
- `workspace.write:path`
- `process.exec:command-class`
- `mcp.server.tool:server/tool`
- `subagent.spawn:profile`
- `network.fetch:host`

## ASK

`ASK` requiere un `PermissionApproverPort` suministrado por surface (CLI/WebUI/ACP). Sin approver, comportamiento seguro = deny explícito, no auto-allow.

Replies posibles inicialmente:

- once;
- session;
- deny.

Persistir approvals solo si el producto necesita policy durable; no mezclar eso con Plugin construction grants.

## Precedence

Evitar reglas mágicas difíciles de auditar. Si se adopta wildcard precedence, documentarla y probarla exhaustivamente. Una opción más simple para Overmind es deny-overrides + specificity explícita; la semántica importa más que copiar `findLast` de OpenCode.

## Security invariant

Plugin grant != model permission.

Un MCP plugin puede recibir internamente un client/process grant para funcionar y, aun así, cada tool external puede requerir PermissionService antes de ejecutarse.
