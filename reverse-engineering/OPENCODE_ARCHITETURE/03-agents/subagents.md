# Subagents

**Status:** VERIFIED-CODE + DERIVED

## Invocation model

Los subagentes se ejecutan mediante `TaskTool`. Desde la perspectiva del modelo parent, delegar es una tool call. Desde la perspectiva del runtime, esa tool crea o usa una session hija y ejecuta un agent distinto con prompt/model/permissions propios.

## Visibility

`ToolRegistry.describeTask(agent)` enumera agents cuyo `mode !== primary` y después filtra los que el parent puede invocar según:

`Permission.evaluate("task", subagent.name, parent.permission)`

Por tanto, que un agent exista no implica que sea visible/invocable desde otro agent.

## Permission boundary

En una subtask se aplica la policy del subagent y se combina con permissions de la session/prompt cuando corresponde. Esto evita tratar el `TaskTool` como bypass de autorización.

Existe además `agent/subagent-permissions.ts`, que centraliza ajustes de permisos en jerarquías de agentes.

## Session hierarchy

El modelo de Session contiene `parentID`. El runtime usa parent/child sessions de forma explícita para task delegation y background work. Una child session es una unidad real de estado: puede tener mensajes, coste, tokens y cancelación relacionados con su parent.

## Cancellation relation

`SessionRunState.cancel` rastrea jobs relacionados usando `sessionId` y `parentSessionId`, por lo que la jerarquía afecta a la propagación de cancelación.

## Architectural implication

Un subagent no es “un prompt auxiliar dentro del mismo turno”. Es una ejecución con identidad y estado propios, encapsulada detrás de una tool del parent. Eso permite:

- contexto especializado;
- policy especializada;
- modelo diferente;
- accounting separado;
- cancelación y persistencia trazables.

## Sources

- `packages/opencode/src/tool/task.ts`
- `packages/opencode/src/session/prompt.ts`
- `packages/opencode/src/tool/registry.ts`
- `packages/opencode/src/agent/subagent-permissions.ts` — `b1b99b48...`
- `packages/opencode/src/session/session.ts`
