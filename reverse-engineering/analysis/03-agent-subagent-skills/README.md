# AGENT 03 — Agents, subagents y skills

## Objetivo

Este análisis reconstruye la evolución del subsistema de agentes de OpenCode a partir de `dev` y de las branches relacionadas con `agent-*`, `agents-*`, `subagent-*`, `skill-*`, `feature/agent-*`, `feature/skill-*` y ramas funcionalmente equivalentes.

La intención no es describir archivos de forma aislada, sino reconstruir **qué abstracciones ejecuta OpenCode, dónde vive el estado, cómo se delega trabajo y qué ideas históricas terminaron integradas, sustituidas o descartadas**.

## Resultado principal

En `dev`, un agente no es un proceso independiente ni un runtime paralelo. Es un **perfil declarativo de ejecución** (`Agent.Info`) que configura prompt, modelo, variante, sampling, límite de pasos y permisos. La unidad real de ejecución y continuidad es `Session`.

Un subagent se materializa cuando `TaskTool`:

1. resuelve un `Agent.Info` de modo no-primary;
2. comprueba profundidad y permiso de delegación;
3. crea o recupera una `Session` hija con `parentID`;
4. deriva sus permisos;
5. selecciona modelo/variant;
6. ejecuta `SessionPrompt` sobre la sesión hija;
7. entrega el resultado al padre en foreground o mediante una notificación sintética en background.

Por tanto, la relación esencial es:

```text
Agent profile
    │
    ├── prompt / model / variant / steps
    └── permission rules
             │
             ▼
Parent Session ── TaskTool ──► Child Session
                                 │
                                 ├── Agent profile
                                 ├── SessionPrompt
                                 ├── Tools
                                 └── Provider / model

Skills ── discovery ──► catalog in system prompt
                         │
                         └── SkillTool ──permission──► full skill content
```

## Documentos

- [`01-runtime-agentes/README.md`](01-runtime-agentes/README.md): modelo de agente vigente y relación Agent–Session–Prompt–Tool.
- [`02-seleccion-modelos-switching/README.md`](02-seleccion-modelos-switching/README.md): agent selection, switching, model routing y variantes.
- [`03-subagents-delegacion/README.md`](03-subagents-delegacion/README.md): spawning, delegación, resume, nesting y comandos.
- [`04-background-observabilidad/README.md`](04-background-observabilidad/README.md): background subagents, notificación, races, ACP y UI.
- [`05-permisos/README.md`](05-permisos/README.md): herencia, aislamiento y límites de privilegios.
- [`06-skills/README.md`](06-skills/README.md): discovery, activación, compatibilidad y evolución v1/v2.
- [`07-prompts-dinamismo-entorno/README.md`](07-prompts-dinamismo-entorno/README.md): prompts, dynamic agents/instructions, references y environment markers.
- [`08-mapa-branches/README.md`](08-mapa-branches/README.md): inventario y agrupación evolutiva de branches.

## Baseline `dev`

Puntos de entrada principales usados como evidencia:

- `packages/opencode/src/agent/agent.ts`
- `packages/opencode/src/agent/subagent-permissions.ts`
- `packages/opencode/src/tool/task.ts`
- `packages/opencode/src/background/job.ts`
- `packages/core/src/background-job.ts`
- `packages/opencode/src/session/system.ts`
- `packages/opencode/src/skill/index.ts`
- `packages/opencode/src/skill/discovery.ts`
- `packages/opencode/src/tool/skill.ts`
- `packages/core/src/v1/config/config.ts`
- `packages/core/src/v1/config/agent.ts`

## Lectura evolutiva de alto nivel

Se distinguen cuatro generaciones funcionales:

1. **Agent profiles + TaskTool**: configuración de agentes primary/subagent y delegación mediante sesiones hijas.
2. **Continuidad y aislamiento**: `subagent-resume`, session IDs, nesting limits, permission inheritance y model selection.
3. **Background/observability**: jobs, notificación al padre, ACP, navegación/UI y tratamiento de carreras.
4. **Skills como contexto lazy**: discovery compatible con `.opencode`, `.claude` y `.agents`, catálogo en system prompt y activación mediante tool. En la línea `core/v2`, las skills evolucionan además hacia eventos/estado de sesión.

## Hechos frente a inferencias

En todos los documentos:

- **Hecho confirmado** significa que existe evidencia directa en código, schema, test o commit.
- **Inferencia** significa que la intención arquitectónica se deduce de cambios convergentes, nombres de branches o secuencias de commits, pero no aparece declarada literalmente.
- **No integrado / alternativo** significa que la branch contiene una idea que no aparece con la misma semántica en `dev` actual.

## Conclusiones rápidas

- La frontera arquitectónica más importante es **Session**, no Agent.
- `TaskTool` es el adaptador que convierte una decisión de delegación del LLM en una nueva ejecución con estado propio.
- Los permisos de un subagent no son una copia completa de los del padre: se heredan restricciones concretas, especialmente `deny`, y el agente hijo conserva su propia policy.
- El modelo de un subagent se decide por prioridad: modelo configurado en el agente; si no existe, modelo del assistant que invoca la task. Varias branches experimentaron con small-model y role routing, pero `dev` no expone ese routing por llamada.
- `task_id` es el identificador de continuidad del subagent en `dev`.
- El background actual usa un registry process-local; no equivale a persistencia durable ni recuperación tras restart.
- Skills y agents son conceptos distintos: un agent define **cómo ejecutar**; una skill aporta **qué instrucciones/contexto especializado cargar**.
- Los environment markers `AGENT=1` / `OPENCODE=1` fueron experimentados en dos líneas, pero no están presentes en el shell vigente de `dev`.
