# OpenCode -> Overmind Extraction Matrix

## Matriz ejecutiva

| Mecanismo OpenCode | Overmind hoy | Decisión | Prioridad |
|---|---|---|---|
| Persistent Session identity | Agent in-memory `_units` | ADAPTAR | P0 |
| Normalized runtime event stream | callbacks + result records; EVENTS deferred | ADAPTAR | P0 |
| RunState / cancellation domain | request-local cancellation | ADAPTAR | P0/P1 |
| Tool execution context | ToolPort simple | ADAPTAR | P1 |
| General Permission engine | construction grants + confinement | ADAPTAR, separado de registry | P1 |
| Child sessions for subagents | deferred contract | ADAPTAR | P1 |
| Background jobs | deferred operational runtime | ADAPTAR tras child sessions | P1/P2 |
| Snapshots/patch/revert | Workspace recovery específica | GENERALIZAR | P1/P2 |
| MCP | future plugin | ADAPTAR como Plugin | P2 |
| Skills | no capability equivalente genérica | ADAPTAR a CONTEXT/TOOLS | P2 |
| Plugin interceptive hooks | composition-only, contracts restrictivos | SOLO hooks tipados mínimos | P2 |
| ACP | no Runtime API todavía | DEFER hasta sessions/events | P2/P3 |
| HTTP server/generated clients | WebUI deferred | DEFER | P3 |
| Large provider registry | explicit ModelTarget mapping | NO COPIAR ahora | — |
| AI SDK/native dual runtime | clean Python ModelBackend seam | NO copiar estructura; sí event vocabulary | — |
| Effect Layers | Python composition root | NO COPIAR | — |
| messages+parts as session history | canonical ContextUnits separados | NO convertir en Context authority | — |
| model-ID tool hacks | canonical tools | NO COPIAR | — |

## P0 — Operational identity before features

El requisito transversal para Memory, WebUI, subagents, background work y durable EVENTS es poder identificar:

- session;
- request/execution;
- model turn;
- tool call;
- runtime event;
- child execution.

Overmind ya tiene `request_id` y `turn_number`, pero son identidad local del Agent. La primera extracción útil es promover identity a contratos runtime estables sin alterar ContextUnit semantics.

## P1 — Safe action runtime

Después de identity/persistence/event ordering:

1. ToolExecutionContext.
2. PermissionService allow/ask/deny.
3. RunState/cancel tree.
4. Child sessions.
5. Mutation journal.

Esta fase convierte Overmind de un agente local con Tools en un runtime capaz de alojar acciones externas, múltiples clientes y delegación segura.

## P2 — Extensibility

MCP, Skills, background services y typed event observers deben construirse sobre los contratos anteriores. De otro modo cada capability inventará su propia identidad, auth, cancellation y telemetry.

## P3 — Interfaces

ACP/HTTP/WebUI deben ser adapters sobre SessionService + EventPort + RuntimeApi. No deberían llamar a Agent internals.

## Diferencia fundamental

OpenCode tiende a usar Session messages/parts como historia persistente y material para el modelo. Overmind debe mantener dos conceptos:

```text
Durable execution record != compiled cognitive context
```

Persistence registra lo ocurrido. ContextCompiler decide lo relevante para pensar ahora.
