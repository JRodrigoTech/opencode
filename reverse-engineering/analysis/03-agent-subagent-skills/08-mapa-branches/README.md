# Mapa evolutivo de branches

## Criterio de clasificación

Se buscaron branches por `agent`, `subagent` y `skill`, y se añadieron ramas funcionalmente relacionadas con background, task lifecycle y protocol observability.

No toda branch cuyo nombre contiene `agent` pertenece al runtime. Se separan:

- **core runtime:** cambia semántica Agent/Session/Task/Skill;
- **evolución v2:** arquitectura alternativa/event-driven en `packages/core`;
- **UI/protocol:** representa o transporta actividad ya existente;
- **infra/no relevante:** nombre incidental, worktree, agentes de reverse engineering o automatización externa.

Además, un tip con merges masivos no se usa como evidencia única del propósito histórico.

## Familia A — Agent definition, selection y switching

| Branch | Clasificación | Hallazgo principal | Estado frente a `dev` |
|---|---|---|---|
| `agent-model-follow` | selection/UI state | TUI sigue modelo configurado por agente y guarda selección por session+agent | problema parcialmente convergente con dev; diseño v2 más explícito |
| `agent-switch-previous` | core/v2 | evento durable de selección incluye agente anterior | arquitectura v2/alternativa |
| `agent-message-previous` | core/v2/session history | causalidad entre mensajes y selección previa | línea v2; inspección puntual recomendada |
| `agent-picker-custom` | UI | agentes custom visibles en selector | concepto integrado |
| `agent-picker-variants` | UI | preview con descripción/model y acceso al model picker | UX, no runtime |
| `run-agent-model` | CLI/core-v2 | selección atómica agent + model policy (`preserve/configured/explicit`) | arquitectura v2; resuelve skew real |
| `agent-settings-skew` | UI/skill skew | tip actual corrige skills sin ID | nombre no representativo del tip |
| `add-dynamic-agents-resolving` | dynamic config | resolución de agentes dinámicos/configurados | concepto de agents custom integrado |
| `test/config-instance-agent-command` | tests/config | interactions de config/agent/command | satélite |

## Familia B — Subagent spawning, resume y limits

| Branch | Clasificación | Hallazgo principal | Estado |
|---|---|---|---|
| `subagent-resume` | runtime | continuar una task previa | integrado conceptualmente como `task_id` |
| `subagent-session-id` | runtime | estabiliza session ID como identidad de task | integrado |
| `v2-subagent-limit` | core/v2 | formaliza nesting limit | integrado conceptualmente como `subagent_depth` |
| `adjust-subagent-tool-description` | prompt/tool UX | ajuste de instrucciones de delegación | satélite al runtime |
| `subagent-command` | commands/runtime | explora commands que disparan subagents | línea experimental |
| `subagent-command-v2` | core/v2 | V2 decide que command no crea child implícito | explícitamente alternativo |
| `task-spec-executor-split` | refactor | separa spec/admission y ejecución de task | evidencia de boundary |
| `task-resume-race` | lifecycle | race de reanudación/promoción/background | concepto absorbido en coordinación de jobs |

### Secuencia reconstruida

```text
Task creates child
     │
     ├─► stable child session ID
     │       │
     │       └─► task_id resume
     │
     ├─► nesting depth
     │
     └─► concurrency/race handling
```

## Familia C — Model routing de subagents

| Branch | Hallazgo | Estado |
|---|---|---|
| `nxl/small-model-subagents` | small model sólo para `explore`, no todos los children | heurística histórica, no baseline actual |
| `subagent-model-routing` | routes `fast`, `smart`, `vision`, `long-context` + exact ID | core/v2, no integrado en `dev` TaskTool |
| `agent-model-follow` | coherencia del modelo al cambiar agente | línea de selección UI/session |
| `run-agent-model` | policy atómica de modelo al seleccionar agent | core/v2 |

### Evolución

```text
heurística small model
        ↓
agent.model declarativo
        ↓
agent/session model-follow
        ↓
role-based ModelRouting (v2 experiment)
```

## Familia D — Permission inheritance

| Branch | Hallazgo | Estado |
|---|---|---|
| `subagent-permission-inheritance` | composición restrictiva parent→child | integrado mediante `deriveSubagentSessionPermission` |
| `agent-permission-order` | precedencia de rulesets | problema real de composición; implementación concreta debe leerse por commit |
| `skill-mention-bypass` | user intent explícito no debe crear bypass global | principio trasladable a agents/tools |

La dirección estable es mantener separadas policy del Agent, restricciones de Session y autorización de la operación.

## Familia E — Background subagents

| Branch | Hallazgo | Estado |
|---|---|---|
| `implement-background-agents` | `background` en TaskTool; resultado sintético al padre | precursor directo |
| `nxl/background-subagents` | registry de jobs, espera de parent idle, rechazo de resume concurrente | precursor/integración parcial |
| `feat/core-v2-background-agent` | lifecycle v2 y problema de correlación activity/result | arquitectura alternativa |
| `subagent-notify` | mueve notificación de TaskTool a SessionPrompt | boundary importante, integrado en línea moderna |
| `subagent-observability` | observabilidad v2 | tip mezclado con merge; usar commits concretos |
| `acp-subagent-events` | expone actividad de child sessions por ACP | integración protocol-specific |

## Familia F — Subagent UI y navegación

Branches:

- `fix/subagent-navigation-inline-click`
- `subagent-card-dnd`
- `subagent-panel-ui`
- `subagent-tab-active`

**Clasificación:** UI/interaction.

No prueban cambios del execution model por sí mismas, pero sí que el `sessionID` del child se usa como identidad navegable y observable en clientes.

## Familia G — Skills v1/lazy

| Branch | Hallazgo | Estado |
|---|---|---|
| `feature/agent-skills` | primera integración agents/skills | origen histórico |
| `feature/skill-tool` | activación explícita mediante tool | diseño integrado en `dev` |
| `migrate-skill-discovery` | discovery multi-source independiente | integrado/evolucionado |
| `read-global-claude-skills` | `~/.claude/skills` global | integrado/compatibilidad |
| `agents-skill-compat` | limita `.claude`/`.agents` a sources de skills | línea core/v2 reciente, boundary claro |
| `styled-skill-errors` | presentación/error UX | satélite |
| `kit/skill-lazy-init` | lazy initialization | relacionada; no usada como evidencia primaria |
| `kit/retrofit-skill-main` | retrofit/refactor | relacionada; requiere análisis granular si se profundiza |
| `kit/httpapi-project-skill-repro` | repro/API | rama diagnóstica |
| `claude/learn-new-skill-Ji0dn` | workflow externo/experimental | no usada para arquitectura core |

## Familia H — Skills v2/session state

| Branch | Hallazgo | Estado |
|---|---|---|
| `fix-skill-mentions` | separa resolución/preparación de skill | core/v2 |
| `skill-mention-bypass` | scoped user-request bypass | core/v2 |
| `skill-source-observer` | observación reactiva de sources | core/v2 |
| `session-skill-activation` | `Skill.Activated` como evento durable de Session | core/v2/futuro alternativo |
| `dev-multi-skills` | múltiples skills inline/chained | feature branch sobre línea dev |
| `agent-settings-skew` | app tolera skill sin ID | compatibilidad UI |
| `effectify-skill` | tip observado sólo lockfile | evidencia insuficiente en tip |

### Dos arquitecturas a no mezclar

```text
DEV actual
Skill metadata -> SystemPrompt -> SkillTool -> conversation context

CORE/V2 branch line
Skill source -> resolve/prepare -> Session.Skill.Activated durable event -> resume/replay
```

## Familia I — Prompts, references y environment

| Branch | Hallazgo | Estado |
|---|---|---|
| `feat/reference-agent` | elimina agente `reference` y absorbe capacidad en `explore` | idea de agent separado descartada |
| `agent-env-markers` | `AGENT=1` / `OPENCODE=1` | no presente en `dev` actual |
| `v2-agent-env` | mismos markers en core shell/PTY con tests | no presente en baseline actual |
| `add-dynamic-agents-resolving` | agentes desde configuración/resolución runtime | custom agents sí están integrados |

## Ramas con `agent` excluidas del análisis funcional principal

### Reverse engineering/worktrees

- `reverse-engineering-agent-01` … `reverse-engineering-agent-10`
- `worktree-agent-a682f34a`
- `worktree-agent-ab6ff98a`
- `worktree-agent-ab5855c0`

Son ramas operativas del entorno de trabajo, no evidencia histórica del producto.

### Automatización/console/infra con nombre incidental

- `chore/duplicate-issues-agent`
- `chore/duplicate-issues-agent-dev`
- `kit/opencode-agent-console-endpoint-backport`
- `kit/opencode-agent-console-endpoint-pin`
- `layer-node-cagent`
- `refactor/core-account-file-agents`

Se excluyen de la reconstrucción central salvo que un análisis específico demuestre que modifican `Agent`, `Session`, `TaskTool` o `Skill`.

## Relación evolutiva global

```text
                     ┌──────────────────────────────┐
                     │ Agent.Info / config profiles │
                     └──────────────┬───────────────┘
                                    │
                      selection/model/switching
                                    │
                                    ▼
                               Session state
                                    │
            ┌───────────────────────┼────────────────────────┐
            │                       │                        │
            ▼                       ▼                        ▼
        TaskTool                SystemPrompt             Skill catalog
            │                       │                        │
            ▼                       │                        ▼
      Child Session                 │                    SkillTool
            │                       │                        │
      resume/background             │                 full instructions
            │                       │                        │
            └──────────────► session lifecycle ◄────────────┘
                                    │
                                    ▼
                        core/v2 durable events
```

## Qué llegó a `dev`

**Confirmado o claramente integrado conceptualmente:**

- custom agent profiles;
- primary/subagent/all modes;
- child sessions para `task`;
- `task_id` resume;
- `subagent_depth`;
- model override por agent + parent fallback;
- permission-aware delegation;
- derivación de permisos parent→child;
- background-job infrastructure experimental;
- skill catalog + lazy SkillTool activation;
- global/project compatibility skill sources.

## Qué permanece alternativo/experimental

- role-based subagent model routing por llamada;
- `session.select` v2 con model policy atómica como contrato principal;
- durable `SessionEvent.Skill.Activated`;
- source observer de skills v2;
- semántica v2 de commands respecto a subagents;
- correlación activity/result formalizada como nueva identidad.

## Qué parece descartado o retirado

- agente dedicado `reference`;
- environment markers `AGENT=1` / `OPENCODE=1` en el baseline actual;
- política general de small-model para todos los subagents;
- spawning implícito de child sólo porque un command seleccione un agent `subagent` en la línea v2.

## Hipótesis arquitectónica final

**Inferencia sustentada por múltiples ramas:** OpenCode converge hacia un diseño en el que:

1. `Agent` es policy declarativa;
2. `Session` es el agregado durable y boundary real de ejecución;
3. `Task/Subagent` es una relación parent-child entre sessions;
4. model selection y permission composition se convierten en policies explícitas;
5. skills evolucionan desde tool output hacia estado/eventos de Session en la arquitectura futura;
6. protocolos y UIs observan la misma topología de sesiones en vez de inventar un runtime paralelo para agents.
