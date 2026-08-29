# Priority Roadmap — valor para Overmind, no paridad con OpenCode

## P0 — Probar delegación de coding con mínima arquitectura

### 1. Congelar invariants existentes

No tocar semántica de ContextCompiler, ModelService, ProtocolUnit o Plugin composition para integrar OpenCode.

### 2. OpenCode capability de alto nivel

Introducir conceptualmente una `OpenCodePlugin`/Tool con una operación tipo:

```text
opencode.delegate(task, session_id?, profile?)
```

El Tool result debe ser bounded y contener la referencia de sesión externa cuando exista.

### 3. Transporte MVP

Usar primero el surface ya existente de OpenCode:

`opencode run --format json --dir <workspace> --agent <restricted-agent> ...`

La production CLI también admite `--session`, `--fork`, `--model` y `--agent`.

**No usar `--auto`, `--yolo` ni `--dangerously-skip-permissions` como política normal del adapter.**

### 4. Dedicated OpenCode agent/profile

Preferir un agent OpenCode específico para Overmind con permisos deliberados en vez de dar a la integración autoridad ilimitada. Overmind controla el workspace autorizado; OpenCode controla sus acciones internas.

**Exit P0:** Overmind puede delegar una misión de software engineering y recibir un resultado bounded sin añadir nuevos subsistemas Core.

## P1 — Enriquecer solo si el uso lo demuestra

### 5. ACP client adapter

Adoptar ACP cuando se necesiten uno o más de:

- session lifecycle estructurado;
- resume/fork estable;
- streaming de agent events;
- cancelación;
- permission request/reply;
- mode/model control.

`opencode acp` ya ofrece estas operaciones sobre stdio.

### 6. Permission forwarding mínimo

Si OpenCode solicita una permission vía ACP, reenviarla a un approver explícito cuando exista. Sin approver, deny. No implementar un PermissionService universal hasta que otra capability de Overmind necesite la misma semántica.

### 7. AgentDelegationPort — condicional

Promover la delegación a un port genérico **solo** cuando aparezca otro agent backend o un subagent nativo con operaciones realmente comunes.

## P2 — Capacidades propias de Overmind

Memory, RAG, Blackboard, MCP, EVENTS, SERVICES, WebUI, persistence y background deben evolucionar por sus propios contracts/requirements de Overmind.

OpenCode puede aportar referencias de implementación, pero no cambia su prioridad automáticamente.

## Triggers para extraer primitives genéricas

| Primitive | Implementar cuando... |
|---|---|
| EventPort | una capability real requiera observación/event facts compartidos, p.ej. Memory/WebUI/services |
| SERVICES lifecycle | un plugin real necesite trabajo determinista persistente/largo, p.ej. ACP process persistente, watcher o MCP connection |
| Session persistence | Overmind necesite resume durable, Memory raw transcript, multi-interface o crash recovery |
| generic PermissionService | dos o más capabilities necesiten runtime approvals/policy común |
| generic SubagentService | existan subagents Overmind o varios agent backends con semántica compartida |
| BackgroundJob | una tarea de Overmind deba sobrevivir al model turn y entregar completion sin polling LLM |
| MutationJournal | múltiples capabilities de Overmind necesiten reconciliation/revert común; no por las mutaciones internas de OpenCode |
| RuntimeApiPort | exista una segunda interaction surface real, p.ej. WebUI |

## Coding-specific features que quedan fuera del roadmap Overmind

No hay fase para reconstruir:

- shell/edit/patch/LSP;
- Git session snapshots de coding;
- coding prompts;
- coding-specific tool routing;
- OpenCode provider compatibility;
- coding subagent topology.

Todo eso puede evolucionar dentro de OpenCode independientemente.

## Resultado estratégico

Overmind gana capacidad de software engineering **sin aumentar proporcionalmente su Core ni su superficie de mantenimiento**.
