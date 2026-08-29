# Acceptance Gates — OpenCode como capability, Overmind intacto

## Core independence gate

- quitar/deshabilitar `OpenCodePlugin` no cambia Agent algorithm.
- Core no importa OpenCode/ACP types.
- ToolRegistry sigue frozen después de READY.
- ContextCompiler sigue siendo único owner de compiled model context.
- ModelService no conoce el external agent.

## Tool protocol gate

- el modelo ve una Tool schema bounded de misión, no el catálogo interno OpenCode.
- solo ToolCalls completas ejecutan delegación.
- el resultado vuelve como ToolExecutionResult/ProtocolUnit normal.
- raw stdout/events no se insertan automáticamente en future context.
- output grande usa summary + references.

## Scope/security gate

- workspace/executable/config sensible se fija por Plugin/config, no por argumento arbitrario del modelo.
- no `--auto`, `--yolo` o `--dangerously-skip-permissions` por defecto.
- subprocess mismo usuario se documenta como no-sandbox.
- secrets no aparecen en Tool result/context.
- selected context compartido es explícito y bounded.

## Failure gate

- executable missing produce error normalizado.
- process crash/timeout no bloquea indefinidamente Runtime.
- malformed JSON/protocol falla explícitamente.
- cancellation no causa automatic full-task retry.
- outcome incierto tras posible side effect se marca como tal.

## Session gate

- external OpenCode session ID se trata como opaque reference.
- resume no importa la transcript completa a Overmind.
- session OpenCode no se confunde con canonical Overmind context/session identity.
- follow-up puede reutilizar external session cuando el transport lo soporte.

## Permission gate para ACP

Cuando ACP sea usado:

- permission request no se auto-aprueba sin policy explícita.
- sin approver, deny/fail closed.
- approved scope no se convierte automáticamente en grant permanente Overmind.
- cancel usa protocol operation cuando sea posible.

## Dedicated-agent gate

- el profile OpenCode usado por Overmind tiene permisos documentados.
- existe al menos una configuración read-only/plan o un rationale explícito si solo hay implement mode.
- OpenCode puede usar sus subagents internamente sin surfaced de toda la child history.

## Context gate

- Provider `messages[]` de Overmind siguen siendo compiled output.
- OpenCode system prompts/reasoning/tool logs no se elevan a system/context authority.
- DelegationResult se somete al mismo ContextCompiler que otras observations.

## Generalization gate

No crear `AgentDelegationPort`, PermissionService, EventPort, SessionService o BackgroundJob solo para satisfacer un diagrama.

Una abstracción genérica entra cuando tiene consumidores reales y tests contractuales que demuestran semántica compartida.

## Definition of done del primer spike

La integración está bien diseñada si Overmind obtiene capacidad útil de coding con **una capability removible**, no si reproduce internamente la arquitectura de OpenCode.
