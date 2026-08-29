# Tool Runtime

**Status:** VERIFIED-CODE

## Two-stage tool construction

OpenCode separa:

1. **Registry**: define qué tools existen y su definición abstracta.
2. **SessionTools.resolve**: decide y adapta qué tools recibe el LLM en esta ejecución.

## Resolve context

Cada tool AI SDK recibe un `Tool.Context` con:

- `sessionID`
- `abort`
- assistant `messageID`
- `callID`
- model
- agent name
- message history
- metadata updater
- `ask()` para permissions
- `extra` con model, bypassAgentCheck y promptOps.

Esto convierte una llamada del LLM en una operación integrada con session state, no una función aislada.

## Schema adaptation

`SessionTools.resolve` toma schemas del registry y aplica `ProviderTransform.schema(model, schema)` para compatibilidad con el provider/model antes de crear el `ai.Tool`.

## Hook lifecycle

Para tools host se ejecutan hooks:

```text
tool.execute.before
    │
    ▼
tool.execute
    │
    ▼
tool.execute.after
```

Los hooks reciben identifiers de session/tool/call y pueden modificar inputs/outputs según contrato.

## Permissions

La tool implementation puede solicitar autorización a través de `ctx.ask`. El runtime combina agent rules y session permissions. El hecho de que una tool esté visible no significa que toda invocación concreta esté autorizada.

## Tool output

La respuesta abstracta soporta:

- `title`
- `output`
- `metadata`
- `attachments`

El processor persiste esos campos dentro del ToolPart completed/error.

## MCP tools

Las tools MCP se convierten también a AI SDK tools. Sus schemas pueden ser transformados y sus resultados se normalizan para el mismo historial de sesión.

## Sources

- `packages/opencode/src/session/tools.ts` — `0f401c7562fa07076afd539990ca12fa207ceee0`
- `packages/opencode/src/tool/tool.ts`
- `packages/opencode/src/tool/registry.ts`
