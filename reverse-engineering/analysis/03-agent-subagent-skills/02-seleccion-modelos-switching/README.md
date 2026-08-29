# Agent selection, model routing y switching

## Baseline `dev`

### Selección de agente

**Hecho confirmado.** `packages/core/src/v1/config/config.ts` define `default_agent`. Debe resolver a un agente `primary`; si no existe o no es válido, el runtime cae a `build`.

Los agentes custom entran en el mismo registro que los built-ins mediante `Agent.state()`. No existe un registry separado para “custom agents”.

### Modelo efectivo

Para una delegación en `packages/opencode/src/tool/task.ts`, la prioridad vigente es conceptualmente:

```text
agent.model
   │ si existe
   ▼
modelo configurado del subagent

si no:

modelo del assistant padre que llamó task
```

La `variant` también forma parte del perfil del agente y de la selección efectiva cuando corresponde.

**Hecho confirmado.** `ConfigAgentV1` permite `model` y `variant`; `TaskTool` resuelve el modelo del agente antes de heredar el modelo del mensaje assistant padre.

## `agent-model-follow`

Commit representativo: `d841feb3e31ae54c4d5b1516a7dac30f17219940` — `fix(tui): follow agent model selection`.

La branch corrige un problema de coherencia en TUI: la selección visual de modelo debía seguir el modelo configurado por el agente, pero también preservar selecciones explícitas por sesión/agente.

Introduce una resolución con candidatos ordenados aproximadamente así:

1. selección explícita guardada para ese agente;
2. modelo durable de la sesión si la sesión ya usa ese agente;
3. modelo configurado en el agente;
4. modelo de la sesión como fallback.

**Hecho confirmado.** La branch crea `resolveAgentModelSelection()` y cambia el estado local de una selección única por sesión a selecciones indexadas por `sessionID + agent`.

**Inferencia.** El problema de fondo era que “agente seleccionado” y “modelo seleccionado” habían evolucionado como estados independientes y podían quedar desalineados al cambiar de agente.

## `agent-switch-previous`

Commit: `717cdeed41662719fb52fc93b7317c96eec8f5ee` — `feat(core): expose previous agent on selection events`.

En la arquitectura `core/v2`, `Session.switchAgent()` publica un evento durable `session.agent.selected`. Esta branch añade el campo `previous`.

```text
session.agent.selected {
  sessionID,
  agent,
  previous?
}
```

**Hecho confirmado.** El valor anterior se toma del estado de la sesión antes de publicar el evento.

**Interpretación.** El switching deja de ser sólo una mutación UI y se convierte en transición observable de estado de sesión. Esto facilita:

- sincronización entre clientes;
- navegación/undo semántico;
- actualización correcta de model selection;
- auditoría/event replay.

Este diseño es más explícito que el baseline v1 de `dev`, aunque `dev` conserva separación conceptual agent/model.

## `run-agent-model`

Commit representativo: `20a953b29cff5a3dc44f00ac547c7455e1ab4f86` — `fix(cli): honor agent model across run targets`.

La branch introduce una API `session.select` capaz de seleccionar agente y política de modelo de forma atómica:

```text
model = preserve
      | configured
      | explicit(Model.Ref)
```

La CLI usa:

- `configured` cuando se especifica agente pero no un modelo explícito;
- `explicit` cuando el usuario especifica ambos;
- `switchModel` cuando sólo se cambia modelo.

**Hecho confirmado.** Esto evita que `opencode run --agent X` conserve accidentalmente un modelo incompatible con la configuración de `X`.

**Inferencia.** `session.select` es una respuesta arquitectónica directa al skew agent/model observado en ramas previas.

## `subagent-model-routing`

Commit: `c71ebe46df0903c0db0aa393e5b168b79cbd4b35` — `feat(core): route subagent models by role`.

Esta branch de la línea `core/v2` introduce `packages/core/src/model-routing.ts` y permite que la invocación del subagent incluya un route opcional:

- `fast`
- `smart`
- `vision`
- `long-context`
- o un ID exacto `provider/model[#variant]`

El resolver filtra modelos activos con tools y text I/O, y ordena candidatos usando capabilities, coste, tags heurísticos, context window y fecha de release.

Prioridad en esa branch:

```text
input.model route
      ↓
resolved routed model
      ↓ si no existe route
agent.model
      ↓ si no existe
parent.model
```

**Hecho confirmado.** El argumento de routing vive en la tool de subagent de `core/v2`.

**No integrado en `dev`.** El `TaskTool` vigente no expone al LLM un selector de roles `fast|smart|vision|long-context` por llamada.

## `nxl/small-model-subagents`

Commit: `ed9621271282cb22e424a563f1d5fd6ce96c31ca` — `fix(task): limit small models to explore agents`.

Esta branch corrige una política previa que enviaba subagents sin modelo explícito al small model del provider. Después del cambio, ese fallback sólo se aplica a `explore`.

**Hecho confirmado.** El diff cambia:

```text
next.model ? undefined : provider.getSmallModel(...)
```

por una condición equivalente a:

```text
!next.model && next.name === "explore"
```

**Lectura evolutiva.** Esta solución heurística antecede a dos estrategias más limpias:

- modelo declarativo por agente;
- routing por roles en `core/v2`.

No debe confundirse con la política vigente de `dev`.

## Agent picker

### `agent-picker-variants`

Commit `16255dd8b6b8fea3fa8f4f59ee1619b2996294da` añade preview del agente en TUI:

- descripción;
- modelo configurado o “uses current session model”;
- acción para saltar al selector de modelo.

### `agent-picker-custom`

Su historial contiene el fix `show selector for custom agents`, confirmando que los agentes custom debían aparecer como ciudadanos de primera clase en la selección.

Estas ramas afectan UX, no cambian el ownership runtime: el agente sigue resolviéndose en el registry/config y la sesión conserva el estado efectivo.

## Modelo conceptual consolidado

```text
                    ┌───────────────┐
                    │ Agent config  │
                    │ model/variant │
                    └───────┬───────┘
                            │
User/session model ─────────┼─────► resolver
                            │          │
Optional route (v2 branch) ─┘          ▼
                                  Model.Ref
                                      │
                                      ▼
                               Session execution
```

## Conclusiones

- **Hecho confirmado:** `dev` resuelve el modelo del subagent desde `agent.model` y después hereda el modelo padre.
- **Hecho confirmado:** varias branches muestran que agent switching y model switching son estados acoplados pero distintos.
- **Hecho confirmado:** `core/v2` avanzó hacia eventos durables y selección agent+model atómica.
- **No integrado en `dev`:** routing LLM-facing por roles (`fast`, `smart`, `vision`, `long-context`).
- **Inferencia:** la dirección arquitectónica es sacar heurísticas ad hoc del `TaskTool` y centralizar selección de modelos en una policy explícita de sesión/model routing.
