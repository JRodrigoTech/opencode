# OpenCode as a Coding Agent Capability for Overmind

**Status:** RECOMMENDED INTEGRATION DIRECTION — not implemented

## Objetivo

Dar a Overmind capacidad avanzada de software engineering **sin convertir su Core ni su WorkspacePlugin en un coding agent**.

OpenCode se ejecuta como agente especializado externo. Overmind decide *qué misión delegar*; OpenCode decide *cómo realizar el trabajo de coding* dentro de su policy.

## Evidencia production

La baseline auditada de OpenCode ofrece dos surfaces relevantes.

### `opencode run`

`packages/opencode/src/cli/cmd/run.ts` documenta un modo non-interactive que envía un prompt, emite eventos y termina cuando la session queda idle. Soporta:

- `--format json`;
- `--continue` / `--session`;
- `--fork`;
- `--agent`;
- `--model`;
- `--variant`;
- `--dir`;
- attachments.

Por tanto puede funcionar como subprocess one-shot/resumable para un MVP.

### `opencode acp`

`packages/opencode/src/cli/cmd/acp.ts` inicia un Agent Client Protocol server por stdin/stdout NDJSON. Internamente levanta OpenCode y construye un ACP Agent sobre su SDK.

`packages/opencode/src/acp/agent.ts` expone explícitamente:

- initialize/authenticate;
- new/load/list/resume/close session;
- fork session;
- session config/mode/model;
- prompt;
- cancel.

`packages/opencode/src/acp/permission.ts` mapea permissions a `requestPermission` con allow once / always / reject y rechaza cuando no existe handler.

Esto convierte ACP en un excelente boundary de integración rica.

## MVP recomendado

### Tool surface

Registrar en Overmind una Tool de alto nivel, no todas las Tools OpenCode:

```text
name: agent.code
arguments:
  task: string
  session_id?: string
  mode?: "build" | "plan" | ...
```

El nombre exacto es producto; el principio es una **misión**, no remote tool passthrough.

### Configuración no model-callable

El workspace/root, executable path, default agent/profile, timeouts y policy no deben ser argumentos arbitrarios del modelo salvo necesidad explícita. Son configuración/grants del Plugin.

### Execution

```text
complete ToolCall
 -> validate bounded task
 -> check OpenCode capability grant
 -> spawn `opencode run --format json --dir <configured-root>`
 -> pass task
 -> consume protocol stdout
 -> keep diagnostics separated
 -> wait/cancel according to Tool boundary
 -> normalize final result
 -> return ToolExecutionResult
```

La output JSON puede utilizarse para observability local, pero no se debe concatenar entera en `content`. Solo el resultado bounded vuelve al model context.

## Resultado

Ejemplo conceptual:

```text
DelegationResult
- status: completed|failed|cancelled
- summary: bounded human/model-readable result
- external_session_id: opaque string
- changed_paths?: bounded list
- verification?: test/build summary
- metadata?: timings / usage refs
```

Outputs grandes deben convertirse en artifact/reference, no en una mega Tool observation.

## Session ownership

El `external_session_id` pertenece a OpenCode. Overmind puede conservarlo en la observation o state privado de la capability para follow-up.

No significa que esa OpenCode session sea una `Session` canónica de Overmind.

```text
Overmind conversation/session (if one exists)
    |
    +-- DelegationRecord -> opencode_session_id="..."
```

Es una referencia cross-boundary, como un job ID o remote thread ID.

## Resume

Un follow-up puede volver a llamar a `agent.code` con el mismo external session ID para mantener el contexto de repository/coding dentro de OpenCode.

Eso evita reenviar a Overmind toda la transcript de la tarea anterior.

## Por qué ACP después del MVP

Subprocess `run` es suficiente para demostrar utilidad. ACP se justifica cuando el adapter necesita semántica interactiva estable:

```text
long-lived OpenCode process
      |
ACP client adapter
      +-> session create/load/resume
      +-> prompt
      +-> event updates
      +-> permission requests
      +-> cancel
      +-> close
```

Entonces el proceso ACP podría ser una futura consumer concreta de Overmind SERVICES lifecycle, pero SERVICES no necesita adelantarse para el spike inicial.

## Permissions

Hay dos autoridades complementarias.

### Outer — Overmind

- existe grant para usar `agent.code`;
- qué workspace se puede delegar;
- qué selected context/artifacts se pueden compartir;
- qué profile externo se permite;
- límites de time/cost cuando sean relevantes.

### Inner — OpenCode

- filesystem operations;
- shell commands;
- web/network;
- external directories;
- internal subagents;
- edit approvals.

ACP permite propagar permission requests. El adapter no debe responder automáticamente `always` salvo policy explícita. Sin approver, deny/fail closed.

## Dedicated agent/profile

Es recomendable configurar en OpenCode un agent dedicado a Overmind, con policy deliberada. No tiene por qué ser `build` unrestricted.

Por ejemplo, distintos perfiles de capability pueden existir del lado OpenCode:

```text
code.inspect      -> read/search/tests, no writes
code.plan         -> analysis/planning, no mutations
code.implement    -> scoped edits + tests
```

Overmind sigue viendo una abstracción de misión; la policy detallada reside donde viven las Tools reales.

## Context sharing

No copiar el ContextFrame completo a OpenCode.

Enviar solo:

- task bien formulada;
- user constraints relevantes;
- optional artifact/file references;
- quizá un compact summary creado explícitamente para la delegación.

Si OpenCode tiene acceso al workspace, debe descubrir source con sus herramientas. Esto aprovecha su specialized context management y reduce duplicación de tokens.

## Result ingestion

El Tool result se almacena como ProtocolUnit normal. Después ContextCompiler decide su inclusión futura exactamente igual que cualquier otra observation.

No elevar automáticamente:

- OpenCode system prompt;
- internal reasoning;
- child-session transcripts;
- shell logs completos;
- raw provider messages.

## Cancellation

MVP subprocess: cancellation termina/interrumpe el proceso según el Tool execution contract y marca outcome; nunca asumir que una side effect incierta no ocurrió.

ACP: usar `cancel` protocol operation. El result debe distinguir `cancelled` de `failed` y registrar incertidumbre/reconciliation cuando corresponda.

## Security

Un subprocess bajo el mismo usuario **no es sandbox**. La documentación de Overmind ya reconoce este punto en `FUTURE/ISOLATION.md`.

Si el coding agent requiere un security boundary más fuerte, el backend de ejecución puede evolucionar a container/sandbox sin cambiar la Tool semantics:

```text
agent.code
 -> OpenCode adapter
 -> process backend
       local | sandbox/container
```

No meter Docker en Core.

## No tool passthrough

No exponer `opencode.read`, `opencode.bash`, `opencode.edit`, etc. uno a uno. Eso convertiría Overmind en coordinador de bajo nivel de un runtime que ya sabe coordinarse por sí mismo.

Preferir:

```text
"Implementa esta feature y verifica los tests"
```

sobre:

```text
read file -> grep -> bash -> edit -> read -> bash ...
```

desde el parent Overmind.

## OpenCode subagents

La misión puede usar `TaskTool` y sus child sessions internamente. Para Overmind, eso es implementation detail del backend especializado.

Solo surfaced metadata realmente útil debe cruzar el boundary — p.ej. final status, artifacts, test result y quizá link/session ID para inspección.

## Acceptance criteria del spike

La integración demuestra su valor si:

1. quitar `OpenCodePlugin` no requiere cambios Core;
2. Agent solo ve una Tool schema bounded;
3. task no recibe filesystem root arbitrario desde el model;
4. OpenCode puede trabajar en el repo configurado;
5. result vuelve como ProtocolUnit normal;
6. raw event stream/transcript no contamina Context;
7. timeout/cancel no redispatcha ciegamente una mutation incierta;
8. permisos peligrosos no se auto-aprueban por defecto;
9. una session OpenCode puede reanudarse sin importar su historia completa a Overmind.
