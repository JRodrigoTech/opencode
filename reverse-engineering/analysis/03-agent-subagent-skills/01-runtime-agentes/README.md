# Runtime de agentes en `dev`

## Tesis

**Hecho confirmado.** En `dev`, `Agent` es principalmente una descripción de política/configuración; no posee un scheduler, event loop ni almacenamiento de sesión propios. La ejecución real ocurre a través de `Session` y `SessionPrompt`.

`packages/opencode/src/agent/agent.ts` construye `Agent.Info` a partir de agentes built-in y configuración de usuario. Entre los campos relevantes se encuentran:

- `name`
- `description`
- `mode`: `primary`, `subagent` o `all`
- `prompt`
- `model`
- `variant`
- `temperature`
- `topP`
- `steps`
- `options`
- `permission`
- `hidden`

El schema de configuración en `packages/core/src/v1/config/agent.ts` confirma además `disable`, `color`, compatibilidad con `maxSteps` y la migración desde `tools` hacia `permission`.

## Built-ins y roles

`Agent.state()` define perfiles nativos y después aplica `cfg.agent`.

Los perfiles visibles en la línea actual incluyen agentes primary y subagents especializados. La distinción `mode` gobierna tanto la selección directa como la elegibilidad para delegación:

```text
primary   -> agente interactivo principal
subagent  -> candidato a TaskTool / @mention
all       -> utilizable en ambos contextos
```

`TaskTool` filtra `Agent.list()` excluyendo `mode === "primary"`; por tanto, la delegación no crea un tipo de entidad diferente: consume el mismo registro de `Agent.Info` con otra política de selección.

## Relación Agent → SessionPrompt

Cuando una sesión se ejecuta, el nombre del agente selecciona:

- prompt adicional del agente;
- model override del agente;
- variant configurada;
- parámetros sampling/options;
- conjunto efectivo de permisos;
- step limit.

`SessionPrompt` combina ese perfil con:

- system prompt específico del provider/model;
- environment/workspace context;
- instrucciones de proyecto;
- skills visibles;
- MCP instructions;
- historial de mensajes;
- tools habilitadas.

**Conclusión:** `Agent` modifica la forma en que una sesión será ejecutada; la sesión sigue siendo el ownership boundary de historial, parent/child relation, metadata y continuidad.

## Agent selection

**Hecho confirmado.** `ConfigV1.Info.default_agent` puede definir el agente primary por defecto. El schema especifica que debe ser primary y que el fallback es `build` si falta o no es válido.

Las surfaces de UI/CLI mantienen una selección de agente asociada a la sesión. Varias branches posteriores, especialmente la línea `core/v2`, intentan hacer esta selección más explícita y atómica con la selección de modelo.

## Agent switching

La historia de branches `agent-message-previous`, `agent-switch-previous`, `agent-picker-custom`, `agent-picker-variants` y `run-agent-model` muestra que el cambio de agente ha tenido tres problemas distintos:

1. recordar cuál era el agente anterior;
2. decidir si el modelo debe preservarse o seguir la configuración del nuevo agente;
3. representar agentes custom en CLI/TUI/app.

En `dev`, los campos de agent/model siguen siendo conceptos separados, aunque el runtime puede resolver el model override del agente cuando la ejecución lo requiere.

## Runtime boundaries

```text
Config
  │
  ▼
Agent.state()
  │  Agent.Info
  ▼
Session selection / message input
  │
  ▼
SessionPrompt
  ├─ SystemPrompt
  ├─ agent.prompt
  ├─ Skill catalog
  ├─ MCP instructions
  ├─ tool registry
  ├─ permission evaluation
  └─ Provider/model execution
```

## Dynamic agents

Branches como `add-dynamic-agents-resolving`, `dynamic-agents-*` y líneas equivalentes indican intentos de hacer la resolución de agentes menos estática y más dependiente de config/runtime.

**Hecho confirmado en `dev`:** agentes definidos en configuración ya se incorporan dinámicamente al registro construido por `Agent.state()`.

**Inferencia:** las branches de dynamic agents no introducen necesariamente una entidad runtime distinta; varias parecen ser etapas de robustecimiento de la resolución de configuraciones, nombres, UI y provider/model binding.

## Agent ↔ tools

Los tools no pertenecen al agente. Se registran globalmente y se filtran/evalúan en contexto de ejecución.

El agente aporta reglas de permiso. Esto es relevante para subagents porque el agente hijo conserva su policy, mientras la sesión hija añade restricciones derivadas del contexto parental.

## Agent ↔ skills

Skills tampoco pertenecen al agente como objetos embebidos. `SystemPrompt.skills(agent)` calcula cuáles son visibles para ese agente usando permisos. La tool `skill` vuelve a evaluar autorización al cargar contenido.

Esto crea una separación útil:

```text
Agent = execution policy
Skill = reusable specialized context
Tool = executable capability
Session = state + continuity
```

## Referencias históricas

La branch `feat/reference-agent` es especialmente reveladora. Una versión inicial creó un agente separado `reference`; un cambio posterior de la misma línea eliminó ese agente y fusionó la capacidad de buscar repositorios referenciados dentro de `explore`.

**Hecho confirmado:** la idea de “reference” como agente dedicado fue retirada en favor de ampliar el prompt/permisos de `explore`.

Esto muestra que los maintainers reservan `Agent` para diferencias de comportamiento suficientemente fuertes; capacidades ortogonales pueden terminar absorbidas por prompts, references o tools sin justificar un agente adicional.

## Inferencias arquitectónicas

- `Agent` está diseñado como una capa de política composable sobre un runtime compartido.
- El sistema evita duplicar loops de ejecución para distintos tipos de agente.
- La diferenciación entre primary/subagent se hace en selección y permisos, no mediante clases runtime separadas.
- La evolución reciente hacia `core/v2` intenta convertir ciertas operaciones de selección y lifecycle en APIs de sesión más explícitas, pero mantiene la misma frontera conceptual: la sesión es la entidad activa.
