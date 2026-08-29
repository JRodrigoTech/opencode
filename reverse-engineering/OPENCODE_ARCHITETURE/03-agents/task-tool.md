# TaskTool

**Status:** VERIFIED-CODE

`TaskTool` es el bridge entre el loop principal y la ejecución de subagents.

## Responsibilities

1. Validar el tipo de subagent solicitado.
2. Obtener definición/config del agent target.
3. Crear o reutilizar la child session apropiada.
4. Construir prompt y contexto para esa child session.
5. Ejecutar el loop del subagent mediante las operaciones de prompt inyectadas (`TaskPromptOps`).
6. Reportar progreso/metadata a la tool call del parent.
7. Devolver el resultado al parent como tool result.

## Why `TaskPromptOps` exists

`SessionTools.resolve` pasa `promptOps` dentro del `Tool.Context.extra`. Esto permite que TaskTool invoque el motor de prompt sin importar directamente todo `SessionPrompt.Service`, reduciendo ciclos de dependencias.

## Dynamic description

La descripción visible de `task` se completa en `ToolRegistry` con el listado de subagents actualmente permitidos. El LLM recibe así un catálogo contextual, no una descripción estática.

## Parent representation

Cuando una subtask viene ya materializada como `SubtaskPart` dentro del loop, `SessionPrompt` crea un `ToolPart` sintético para `task`, lo marca running, ejecuta hooks, invoca TaskTool y completa/falla el mismo part. La delegación queda por tanto preservada en el historial del parent.

## Sources

- `packages/opencode/src/tool/task.ts`
- `packages/opencode/src/session/tools.ts`
- `packages/opencode/src/session/prompt.ts`
- `packages/opencode/src/tool/registry.ts`
