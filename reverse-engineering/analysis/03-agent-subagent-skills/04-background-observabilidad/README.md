# Background subagents y observabilidad

## Problema arquitectónico

Ejecutar un subagent en foreground es sencillo: la `task` espera a que termine la sesión hija y devuelve el texto. Background introduce tres problemas adicionales:

1. desacoplar la vida del child de la llamada a la tool;
2. correlacionar el resultado posterior con la sesión padre correcta;
3. decidir cuándo reactivar/notificar al padre sin crear loops o duplicados.

Las branches muestran una evolución explícita de estas responsabilidades.

## Primera generación: `implement-background-agents`

En `packages/opencode/src/tool/task.ts` de esta branch aparece `background?: boolean`.

Cuando es `true`:

- la tool arranca `run()` sin bloquear;
- devuelve inmediatamente un estado `running` con `task_id`;
- al terminar, inyecta en el padre un mensaje sintético `noReply` con `task_result` o `task_error`;
- intenta continuar el padre sólo si sigue siendo seguro hacerlo.

**Hecho confirmado.** La implementación incluye lógica `latestUser`, detección de polling mediante `task_status` y `continueParent`.

**Limitación.** La responsabilidad de notificar al padre está embebida dentro de `TaskTool`, por lo que el lifecycle de “una sesión hija terminó” no es genérico.

## `nxl/background-subagents`

Commit `45360e5e0bba73b40b4c982df778d60902cf4576` — `fix(task): handle running background resumes`.

La branch corrige dos races importantes:

- rechaza reanudar una task cuyo job ya está `running`;
- si el padre está ocupado al terminar el child, espera al evento `SessionStatus.Idle` y vuelve a intentar la continuación.

También usa `BackgroundJob.Service` como registry de ejecución.

**Hecho confirmado.** El código espera el idle del padre con subscription/event stream y limita los intentos.

### Implicación

La semántica background ya no puede modelarse sólo como `Promise` fire-and-forget. Necesita estado consultable y coordinación con el state machine de la sesión padre.

## `task-resume-race`

Esta familia aborda la colisión entre:

- reanudar una sesión hija como foreground;
- una ejecución background que todavía está registrada/running;
- promoción o transición entre ambos modos.

**Hecho confirmado por la evolución posterior:** el runtime actual consulta el registry de background y trata explícitamente los jobs activos.

**Inferencia:** el ID estable de sesión (`task_id`) hizo inevitable resolver exclusión mutua/lifecycle; dos ejecuciones concurrentes sobre la misma child session romperían ordering y causalidad del historial.

## `feat/core-v2-background-agent`

Commit representativo `dc02446ff4adc1568404a59a1d8d28c23f89c292` documenta una frontera de correlación de resultados.

El comentario de código explica que un simple admission ID no basta para asociar la respuesta cuando un drain de sesión puede procesar trabajo posterior encolado. Se necesita una identidad acotada de activity/result.

**Hecho confirmado.** La preocupación aparece directamente en `packages/core/src/tool/task.ts`.

**Importancia.** Esto revela una constraint de arquitectura event-driven: `sessionID` identifica el agregado, pero no necesariamente una ejecución concreta dentro de ese agregado.

## `subagent-notify`: mover lifecycle a Session

Commit `d5ed132cd83b232837f7888b6a57efeac400e670` — `fix(session): notify parent when subagents finish`.

Esta branch mueve gran parte de la notificación desde `TaskTool` a `SessionPrompt`.

`SessionPrompt.notifyParent()`:

- comprueba `parentID`;
- evita duplicados mediante `subagent_last_notified_message_id`;
- consulta `BackgroundJob` para distinguir foreground/background;
- carga la sesión padre;
- transforma el resultado del child a un mensaje sintético;
- lo inserta en el padre.

**Hecho confirmado.** `TaskTool` elimina decenas de líneas de lógica específica de inyección y el servicio `SessionPrompt` adquiere `BackgroundJob` como dependencia.

### Interpretación

Este cambio convierte:

```text
TaskTool knows how to notify parent
```

en:

```text
Session lifecycle knows child completion semantics
```

Es una mejora de boundary: cualquier child session que finalice puede compartir la misma semántica de notificación.

## `BackgroundJob` en `dev`

`packages/opencode/src/background/job.ts` expone el servicio respaldado por `packages/core/src/background-job.ts`.

**Hecho confirmado.** El registry es process-local/in-memory.

Consecuencias:

- permite `start/get/wait` durante la vida del proceso;
- evita doble ejecución mientras el proceso vive;
- **no** proporciona recuperación durable después de restart;
- la persistencia de la child `Session` no implica persistencia del execution job.

Es importante no describir background subagents como un job queue durable.

## Feature flag en `dev`

**Hecho confirmado.** La ejecución background actual está detrás de `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS`/infraestructura experimental equivalente en el `TaskTool` vigente.

Por tanto, el código existe en baseline, pero no debe interpretarse como contrato estable universal.

## `acp-subagent-events`

Commit `ca357ee2a0e1ac363d03b9017181af7f842e9409` — `fix(acp): surface subagent activity`.

La integración ACP añade observabilidad de children:

- mantiene un map de child sessions;
- registra `parentID`, `rootID`, `depth` y title;
- resuelve eventos originados en una child hacia la sesión ACP root;
- prefija tool calls/permissions para evitar colisiones;
- reenvía chunks de mensaje, thoughts y tool updates de subagents.

**Hecho confirmado.** La branch escucha `session.created`/`session.deleted` y adapta eventos de children.

### Implicación

Un protocolo externo no puede tratar la sesión raíz como único productor de eventos si los subagents son sesiones reales. Debe preservar causalidad/source session aunque presente una vista unificada.

## `subagent-observability`

El tip observado contiene un merge amplio desde `v2`, por lo que no es fiable atribuir todos sus cambios al feature por nombre.

**Clasificación:** línea de observabilidad v2 relacionada, pero cualquier afirmación de implementación debe apoyarse en commits individuales, no en el diff completo del tip.

## Ramas UI satélite

- `fix/subagent-navigation-inline-click`
- `subagent-card-dnd`
- `subagent-panel-ui`
- `subagent-tab-active`

Estas ramas revelan que la child session se convirtió también en entidad navegable/visualizable en clientes.

**Hecho estructural inferido con alta confianza:** una vez el subagent tiene `sessionID` propio, la UI puede tratarlo como conversación navegable; no hace falta inventar un modelo de “task transcript” separado.

No se consideran cambios del execution model salvo que un commit específico toque runtime/session.

## State machine conceptual

```text
created
   │
   ▼
running ────────┐
   │            │ resume while running -> reject/coordinate
   ├─foreground │
   │   wait     │
   │            │
   └─background ┘
        │
        ├─ completed
        └─ error
             │
             ▼
      notify/inject parent
             │
             ▼
     optional parent resume
```

## Conclusiones

- Background no es simplemente “no await”; requiere job identity, state, parent coordination y deduplication.
- La evolución mueve responsabilidad desde `TaskTool` hacia services de Session/lifecycle.
- ACP confirma que subagents son fuentes de eventos de primera clase.
- `BackgroundJob` vigente es process-local, no una cola durable.
- La principal deuda visible es la correlación entre admission/work item/result dentro de una sesión con trabajo potencialmente encolado.
