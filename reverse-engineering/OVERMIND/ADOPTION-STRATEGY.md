# Adoption Strategy — usar OpenCode sin absorber OpenCode

## Principio

OpenCode debe aportar **evidencia de soluciones maduras** y, cuando corresponda, actuar como **agente especializado invocable**. No debe convertirse en el blueprint arquitectónico de Overmind.

La pregunta correcta no es:

> ¿Cómo implementamos en Overmind lo que hace OpenCode?

Sino:

> ¿Necesita Overmind esta capacidad universalmente, o puede consumirla como una capacidad especializada externa?

## Árbol de decisión

```text
Mecanismo observado en OpenCode
        |
        +-- ¿Es específico de software engineering?
        |       |
        |       +-- sí -> DELEGATE-TO-OPENCODE
        |       |
        |       +-- no
        |
        +-- ¿Overmind ya tiene un boundary equivalente y mejor alineado?
        |       |
        |       +-- sí -> PRESERVE-OVERMIND
        |       |
        |       +-- no
        |
        +-- ¿Existe una capability real de Overmind que lo necesita ahora?
                |
                +-- no -> REFERENCE-ONLY / DEFER
                |
                +-- sí -> adaptar la primitive mínima
```

## Regla de promoción a Core

Una necesidad de una sola capability no justifica automáticamente una abstracción Core.

Preferencia:

1. mantener la implementación dentro del Plugin/capability que la necesita;
2. estabilizar el contrato y obtener evidencia;
3. promover una primitive neutral a Core cuando **dos o más capacidades independientes** necesiten la misma semántica universal o cuando el Core deba gobernarla por autoridad/seguridad.

Esto está alineado con la filosofía actual de Overmind: Core pequeño, simplify first, public ports explícitos y capabilities removibles.

## Coding como ejemplo canónico

Un coding agent requiere muchas decisiones de dominio:

- navegación de repositorio;
- shell y command execution;
- edit/patch strategies;
- LSP;
- Git state/revert;
- prompts de coding;
- model-specific tool behavior;
- subagents de exploración/implementación;
- permisos específicos de filesystem/shell.

OpenCode ya posee ese runtime. Construir una segunda versión dentro de Overmind duplicaría complejidad y obligaría a Overmind Core a conocer un dominio que no es universal.

La adaptación correcta es:

```text
Overmind Agent
 -> high-level delegation Tool
 -> OpenCode adapter
 -> OpenCode agent
 -> bounded result
 -> Overmind ProtocolUnit
```

## Tool primero, framework después

La primera integración de un agente externo no necesita un `SubagentService`, un EventBus global ni un SessionService nuevo.

Si una sola `OpenCodePlugin` puede registrar una Tool que resuelva el caso de uso de forma segura, ese es el MVP correcto.

Solo si aparecen más backends de agentes o se necesitan resume/background/cancel genéricos conviene extraer un `AgentDelegationPort` común.

## Boundary de contexto

Overmind no debe entregar automáticamente todo su ContextFrame a OpenCode.

La delegación recibe un paquete explícito y bounded:

```text
DelegationInput
- task
- workspace/capability scope
- user constraints
- optional selected artifacts/context
- optional external session reference
```

Para coding suele ser preferible dejar que OpenCode descubra el repositorio mediante sus propias tools en lugar de copiar grandes cantidades de source a la ventana de Overmind y después reenviarlas.

El resultado vuelve también bounded:

```text
DelegationResult
- status
- summary
- external_session_id
- artifact/change references
- test/verification summary
- error if any
```

La transcript completa del agente externo no se convierte automáticamente en contexto futuro.

## Authority y permisos en dos niveles

### Overmind

Decide si se permite delegar, a qué capability, con qué workspace/scope y qué información se comparte.

### OpenCode

Decide y aplica sus permisos internos de read/edit/shell/web/subagents durante la ejecución de coding.

No hace falta copiar el permission engine de OpenCode para conservar esa separación. Si la integración ACP produce permission requests, el adapter puede reenviarlas a un approver explícito; sin approver debe fallar cerrado.

## Ownership de modelos

Overmind mantiene su `ModelService`. OpenCode mantiene su provider/model runtime para la misión delegada.

No existe motivo para que Overmind replique el provider registry de OpenCode ni para que OpenCode se convierta en un ModelBackend de Overmind: es un **agent capability**, no una simple inferencia.

## Independence invariant

Eliminar OpenCode del entorno debe quitar la capacidad de coding delegada, pero no romper:

- Agent core;
- ContextCompiler;
- ModelService;
- Tool protocol;
- otros Plugins.

Esa removibilidad es la prueba principal de que la integración no ha contaminado la arquitectura de Overmind.
