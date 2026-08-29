# Subagents

**Status:** VERIFIED-CODE + DERIVED

## Invocation model

Los subagentes se ejecutan mediante `TaskTool`. Desde el modelo parent, delegar es una tool call. En runtime, `TaskTool` crea o reanuda una **child session** con agent, model, permisos e historia propios y la ejecuta mediante las mismas operaciones de prompt del host.

## Visibility

`ToolRegistry.describeTask(agent)` enumera agents cuyo `mode !== "primary"` y filtra según:

`Permission.evaluate("task", subagent.name, parent.permission)`.

Que un agent exista no implica que sea visible/invocable desde otro agent.

## Child session and resume

`TaskTool` acepta `task_id`. Si referencia una session existente, se reanuda esa child session; si no, crea una nueva con:

- `parentID = parent session`;
- título derivado de la descripción;
- agent target;
- ruleset derivado del parent/subagent;
- denies adicionales para tools que no deben heredarse implícitamente.

El resultado devuelve el `task_id`/session ID dentro del envelope de task, permitiendo continuidad explícita.

## Depth limit

Antes de crear el hijo se recorre `parentID` hacia arriba y se calcula depth. Si alcanza `config.subagent_depth` (default observado: `1`) falla con `Subagent depth limit reached`.

La delegación recursiva está por tanto acotada por policy de configuración.

## Model inheritance

El child usa `next.model` cuando el subagent tiene model propio. Si no, hereda model/provider del assistant parent y conserva la variant del parent cuando corresponde.

## Permission boundary

La child session usa `deriveSubagentSessionPermission()` y se combina con reglas propias del agent. La ejecución no trata `TaskTool` como bypass de autorización. La propia invocación `task:<subagent>` pasa por `ctx.ask()` salvo el camino interno `bypassAgentCheck`.

## Foreground vs background

La misma ejecución puede vivir en `BackgroundJob`.

- foreground espera el job o su promoción;
- `background=true` requiere `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true`;
- background retorna inmediatamente un envelope `state="running"`;
- al terminar, el runtime inyecta un **prompt sintético** en la parent session con el resultado/error;
- si una task existente sigue activa, una llamada con el mismo child ID puede extenderla con contexto adicional.

Esto separa identity/lifecycle de la tool call que la inició.

## Cancellation

El job del child instala `onInterrupt(() => ops.cancel(childSession))`. El foreground también enlaza el abort signal del tool context con `cancel(childSession)`. `SessionRunState` amplía esa semántica cancelando BackgroundJobs relacionados por `sessionId` y `parentSessionId`.

## Architectural implication

Un subagent en OpenCode no es un prompt auxiliar dentro del mismo context window. Es una ejecución identificable, persistible y reanudable detrás de una tool del parent. Esta separación habilita contexto, model routing, policy, accounting y cancelación independientes.

## Sources

- `packages/opencode/src/tool/task.ts` — `d8ca640cfba9a52d97e5180fda0ffa719910592b`
- `packages/opencode/src/session/prompt.ts` — `0f85d44f209ba792065aeb951f0bd2e12b59fae8`
- `packages/opencode/src/tool/registry.ts` — `9167cb3ea6bc5c8dd075f0f8271adbdec6074b12`
- `packages/opencode/src/agent/subagent-permissions.ts` — `b1b99b484e76b4d363c5a3af656723ae905dad8a`
- `packages/opencode/src/session/run-state.ts` — `5cefdd04a3f3bb712c67c1f89687644a633509d0`
