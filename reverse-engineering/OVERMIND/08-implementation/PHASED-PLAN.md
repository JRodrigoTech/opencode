# Phased Plan — integración primero, abstracciones después

## Phase A — Preserve invariants

1. Mantener ContextCompiler y ContextUnits sin cambios semánticos.
2. Mantener Tool protocol fail-closed y ProtocolUnit atomicity.
3. Mantener Plugin stage/validate/freeze.
4. No introducir Session/Event/Permission frameworks para preparar la integración.

**Exit:** baseline de Overmind permanece estable.

## Phase B — OpenCode delegation spike

1. First-party `OpenCodePlugin` conceptual.
2. Una Tool de alto nivel `agent.code`/equivalente.
3. Configuración fija/scoped de executable y workspace.
4. Subprocess `opencode run --format json` como primer transport.
5. Parsear output protocol sin volcarlo completo al Context.
6. Devolver `DelegationResult` bounded con external session ID cuando sea posible.
7. Timeout/cancellation conservador; no blind retry de mutaciones inciertas.

**Exit:** una misión de coding real puede completarse sin modificar Agent/Core.

## Phase C — Harden capability boundary

1. Dedicated OpenCode agent/profile con least privilege.
2. Input/output size limits.
3. Data-sharing policy explícita.
4. error taxonomy del adapter.
5. tests de process failure, malformed protocol y uncertain side effects.
6. verify plugin removal leaves Core unchanged.

**Exit:** la capability es segura, removible y testeable como Plugin independiente.

## Phase D — ACP, only if required

Migrar o añadir `OpenCodeAcpAdapter` si el uso demuestra necesidad de:

- persistent external sessions;
- structured resume/fork;
- permission forwarding;
- interactive cancellation;
- richer progress;
- process reuse.

Un proceso ACP long-lived puede ser el primer caso concreto para futura SERVICES lifecycle.

**Exit:** lifecycle rico sin filtrar wire protocol a Agent/Core.

## Phase E — Generic delegation, only with second consumer

Si aparece otro external agent o un subagent nativo Overmind con semántica común:

1. comparar contratos reales;
2. extraer `AgentDelegationPort` mínimo;
3. mantener adapters privados por backend;
4. no forzar lowest-common-denominator si las capacidades son distintas.

## Independent Overmind evolution

Memory, RAG, Blackboard, persistence, EVENTS, SERVICES, MCP, WebUI y native subagents siguen su roadmap propio. OpenCode sirve como referencia cuando estos subsistemas lleguen, pero no se convierten en fases obligatorias de esta integración.

## No phase for coding-stack duplication

No existe fase para implementar en Overmind shell/edit/LSP/Git/coding prompts/code-mode. Ese dominio permanece detrás de OpenCode.
