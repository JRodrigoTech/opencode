# Subagents: spawning, delegación, resume y continuidad

## Modelo vigente en `dev`

**Hecho confirmado.** Un subagent no es una entidad de persistencia diferente de una sesión. `packages/opencode/src/tool/task.ts` implementa la delegación creando o recuperando una `Session` hija.

Flujo reconstruido:

```text
LLM llama task
   │
   ├─ subagent_type
   ├─ prompt
   ├─ description
   ├─ task_id?       -> continuación
   └─ background?    -> ejecución experimental background
   │
   ▼
TaskTool
   ├─ lista agentes mode != primary
   ├─ filtra por permiso task
   ├─ valida profundidad
   ├─ resuelve Agent.Info
   ├─ crea/reutiliza Session hija
   ├─ deriva permisos de sesión
   ├─ resuelve modelo
   └─ ejecuta SessionPrompt
```

## Selección de subagent

`TaskTool` construye su descripción usando los agentes cuyo `mode !== "primary"`. Si existe un agente caller, la lista que se muestra al modelo se filtra además por la evaluación del permiso `task` para el nombre del candidato.

Esto importa porque hay dos capas de control:

1. **discovery/admissibility:** qué subagents ve el modelo;
2. **execution authorization:** comprobación de permiso al ejecutar la tool.

**Hecho confirmado.** Invocaciones explícitas del usuario mediante ciertas surfaces pueden usar `bypassAgentCheck`, evitando una segunda aprobación de selección del agente; esto no equivale a desactivar el resto de permisos del subagent.

## Spawning

Si no se proporciona `task_id`, `TaskTool` crea una sesión hija con `parentID = ctx.sessionID`.

La sesión hija obtiene:

- título asociado a la tarea/subagent;
- relación parent/child;
- policy de permisos derivada;
- agente seleccionado;
- modelo efectivo;
- historial independiente.

La ejecución del prompt ocurre sobre el ID de esa sesión hija, no sobre la sesión del padre.

### Consecuencia arquitectónica

```text
Parent Session
   │
   ├─ assistant message
   │      │
   │      └─ tool call: task
   │
   └──────────────► Child Session
                       ├─ parentID
                       ├─ own messages
                       ├─ selected agent
                       └─ own execution loop
```

El aislamiento de historial es, por tanto, una propiedad de sesiones, no de un scheduler especial de subagents.

## Resume / continuation

### `task_id`

**Hecho confirmado en `dev`.** `task_id` indica que una task previa debe continuar en la misma sesión hija. `TaskTool` intenta recuperar la sesión por ese ID; si existe, la reutiliza.

Esto preserva:

- historial previo del subagent;
- contexto ya producido;
- identidad de la tarea;
- relación con el padre.

### Evolución histórica

La familia relevante incluye:

- `subagent-resume`
- `subagent-session-id`
- `v2-subagent-limit`
- `task-resume-race`
- `nxl/background-subagents`

**Lectura consolidada:** la continuidad evolucionó desde “crear una nueva tarea” hacia “identificar una conversación hija durable y poder reentrar en ella”.

`subagent-session-id` representa la estabilización del ID de sesión como contrato de la task; `task_id` deja de ser un identificador efímero de una ejecución y actúa como identidad de continuidad.

## Límite de nesting

`packages/core/src/v1/config/config.ts` define `subagent_depth`:

> Maximum subagent nesting depth. Defaults to 1, which prevents subagents from launching subagents.

**Hecho confirmado.** El valor es configurable y no negativo.

Semánticamente:

```text
root session depth = 0
child            = 1
child-of-child   = 2
...
```

El default `1` permite delegación desde una sesión principal, pero impide que el subagent vuelva a delegar.

### `v2-subagent-limit`

Esta branch pertenece a la formalización de ese límite en la arquitectura v2. Su existencia y posterior presencia del campo `subagent_depth` en el schema de `dev` indican integración conceptual.

**Inferencia controlada.** Aunque la implementación concreta cambió entre core/v2 y opencode/v1, el boundary de seguridad —limitar recursion de agents— sobrevivió.

## Commands y subagents

Las ramas `subagent-command` y `subagent-command-v2` son importantes porque exploran si ejecutar un command con un agente `subagent` debe crear automáticamente una sesión hija.

### Resultado de `subagent-command-v2`

Commit `b5cf9081c9c5d4d232264f62517bb3d2a8a774ab` — `refactor(core): keep subagent commands focused`.

La documentación de esa branch se corrige explícitamente para establecer que en V2:

- `subtask` se acepta por compatibilidad, pero se ignora;
- un command corre en la sesión actual;
- seleccionar un agente cuyo modo es `subagent` **no** convierte el command en subtask.

**Hecho confirmado.** También se elimina el `agent = "general"` implícito del command `review`.

### Interpretación

Se separan dos conceptos:

```text
Command = prompt/template invocation in a session
Task    = explicit delegation boundary creating child session
```

**Inferencia.** Esta separación reduce efectos implícitos: la creación de un subagent debe ser una operación explícita de delegación, no una consecuencia accidental del `mode` del agente seleccionado por un command.

## `task-spec-executor-split`

Esta familia intenta separar la especificación declarativa de la task de su ejecutor/lifecycle.

**Lectura arquitectónica.** Es coherente con la evolución general: `TaskTool` concentra demasiadas responsabilidades —admission, child creation, permissions, model resolution, foreground/background lifecycle, result formatting— y varias branches extraen responsabilidades hacia Session, BackgroundJob o plugins.

## Cancelación foreground

En foreground, la ejecución del child queda ligada al abort de la tool/padre. Históricamente, tests de la línea background/core-v2 verifican que interrumpir la espera foreground interrumpe el child.

Esto distingue dos lifecycles:

- **foreground:** el caller espera el resultado y la cancelación se propaga;
- **background:** el child debe sobrevivir a la finalización inmediata de la tool y notificar después.

## Resultado de la task

El contrato histórico y actual devuelve el identificador de la sesión hija junto con el resultado, permitiendo reanudación posterior.

La semántica importante no es el formato textual exacto, sino:

```text
result = {
  child session identity,
  task output,
  metadata for UI/runtime
}
```

## Conclusiones

- **Hecho confirmado:** `TaskTool` es el verdadero spawning boundary.
- **Hecho confirmado:** el subagent es una `Session` hija, no un actor/runtime separado.
- **Hecho confirmado:** `task_id` implementa continuidad reutilizando la misma sesión hija.
- **Hecho confirmado:** `subagent_depth` limita recursión; default 1.
- **Hecho confirmado en V2:** commands y subagent spawning se desacoplaron deliberadamente.
- **Inferencia:** las branches muestran una dirección hacia mover lifecycle genérico fuera de `TaskTool` y dejar a la tool como adaptador de delegación/admission.
