# Permissions by Agent

**Status:** VERIFIED-CODE

## Evaluation algorithm

La policy efectiva no se calcula con una tabla fija por agente. Se fusionan rulesets y `Permission.evaluate(permission, pattern, ...rulesets)` hace:

1. `flat()` de rulesets en orden;
2. busca desde el final (`findLast`);
3. require match wildcard tanto de permission como pattern;
4. si no hay match, default `ask`.

## Consequence

Una regla posterior más específica o incluso más general puede sobrescribir una anterior. Auditar permisos exige conservar el orden exacto.

## Logical permission aliases

Para decidir visibilidad global de tools:

- `edit`, `write`, `apply_patch` se agrupan bajo permission `edit`;
- `list_mcp_resources`, `list_mcp_resource_templates`, `read_mcp_resource` bajo `read`.

Esto permite ocultar familias de tools con una sola regla wildcard.

## Agent → Task permission

La visibilidad de subagents usa `task` como permission y el nombre del agent target como pattern. Así puede permitirse `task:explore` y denegarse otro subagent.

## `ask` lifecycle

Si una pattern es deny, `ask()` falla inmediatamente. Si todas son allow, no crea request. Si alguna queda ask:

- crea `PermissionV1.Request`;
- publica `Permission.Asked`;
- espera un `Deferred`.

Replies:

- `reject`: falla request actual y las pendientes de la misma session;
- `once`: permite solo la actual;
- `always`: permite actual y añade rules allow a `state.approved` para los patterns `always`.

Las nuevas approvals pueden resolver requests pendientes de la misma session.

## Persistence boundary

Las approvals `always` del `Permission.Service` son instance-state runtime. La session, por separado, puede persistir un ruleset `permission` que se combina al resolver tools/system MCP. No deben confundirse ambos conceptos.

## Sources

- `packages/opencode/src/permission/index.ts` — `2e27ff2424dbb000ea9ed7f73471769716ba40a1`
- `packages/opencode/src/agent/agent.ts`
