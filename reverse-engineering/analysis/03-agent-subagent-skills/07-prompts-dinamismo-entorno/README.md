# Prompts, dynamic agents, references y environment markers

## Composición del contexto del agente

En `dev`, la identidad práctica de un agente emerge de varias capas, no de un único prompt estático.

```text
provider/model system prompt
        +
environment/workspace context
        +
agent.prompt
        +
project/user instructions
        +
skill catalog
        +
MCP instructions
        +
tools + permissions
        +
session history
```

`packages/opencode/src/session/system.ts` selecciona prompts base por familia de modelo/provider y añade información del entorno.

## Environment prompt vigente

**Hecho confirmado.** `SystemPrompt.environment(model)` incluye:

- nombre/model ID exacto;
- working directory;
- workspace/worktree root;
- si el directorio pertenece a un repositorio git;
- plataforma;
- fecha;
- referencias de proyecto disponibles con name/path/description.

Esto significa que “environment awareness” es contexto del system prompt, no estado propio de `Agent.Info`.

## `agent.prompt`

Cada agente puede añadir un prompt especializado. Built-ins como `explore`, `compaction`, `title` o perfiles primarios se diferencian en buena medida por:

- prompt;
- permisos;
- model/variant/options;
- mode.

**Inferencia:** OpenCode prefiere construir especialización como policy + prompt sobre un runtime común antes que crear clases de agente distintas.

## Dynamic agents

### Baseline

`Agent.state()` combina agentes nativos y entradas de `cfg.agent`.

**Hecho confirmado.** Los agentes custom son dinámicos respecto a la configuración cargada. Pueden definir descripción, prompt, model, variant, mode, hidden, steps, permissions y options.

### `add-dynamic-agents-resolving`

Esta familia está asociada a robustecer resolución dinámica de agentes en la arquitectura más reciente.

**Clasificación:** evolución de lookup/config binding, no evidencia de un scheduler dinámico o de generación autónoma de agentes.

El término “dynamic agent” debe interpretarse aquí como **agente definido/resuelto en runtime desde configuración**, salvo evidencia específica de una branch que cree perfiles generados durante una sesión.

## References y evolución de `reference` agent

La branch `feat/reference-agent` es una evidencia especialmente clara de boundary correction.

Una implementación anterior tenía un agente `reference` separado para buscar repositorios externos. El commit `0d2bd182b1eb82c10f5603e935e542d0e7377a67` elimina ese agente y:

- añade referencias válidas al prompt de `explore`;
- permite a `explore` leer bajo el path global de references;
- extiende su descripción para indicar cuándo buscar en proyectos referenciados;
- elimina `prompt/reference.txt`.

**Hecho confirmado.** La capacidad “consultar referenced projects” se absorbió dentro de `explore`.

### Conclusión de diseño

No toda fuente de contexto merece un Agent nuevo.

```text
antes:
Reference Agent -> external repos

posteriormente:
Explore Agent + available references -> external repos
```

La especialización se mantuvo como contexto/permisos sobre un agente existente.

## Skills dentro del prompt

`SystemPrompt.skills(agent)` anuncia sólo las skills visibles según permisos y pide usar `SkillTool` para cargar su contenido.

Por tanto, el agent prompt no incorpora de forma eager todos los conocimientos especializados.

Esta separación evita que un agente con muchas skills disponibles tenga un system prompt proporcionalmente enorme.

## Agent ↔ MCP

`SystemPrompt.mcp(agent, permission?)` filtra instrucciones MCP en función del ruleset efectivo.

**Hecho confirmado.** La visibilidad del contexto de integración también depende de permisos: si ninguna tool asociada a una instrucción MCP está disponible, esa instrucción puede omitirse.

Esto revela un principio general:

> OpenCode intenta que el prompt describa sólo capacidades que la ejecución realmente puede ejercer.

## Environment markers: `AGENT=1` y `OPENCODE=1`

Dos líneas experimentaron con variables de entorno para que procesos lanzados por OpenCode pudieran detectar que se ejecutaban bajo un agente.

### `v2-agent-env`

Commit `576b660d3837e5dc50555cb0e58d3bac00e11e07` — `fix(core): mark user processes as opencode agents`.

Añade a shell y PTY:

```text
AGENT=1
OPENCODE=1
```

También añade tests que esperan `1|1` desde procesos shell/PTY.

### `agent-env-markers`

Es una línea equivalente en la arquitectura opencode/v1.

### Comparación con `dev`

Se inspeccionaron:

- `packages/opencode/src/tool/shell.ts`
- `packages/core/src/tool/bash.ts`

**Hecho confirmado:** el baseline `dev` actual no establece esos dos markers en esas rutas.

**Clasificación:** idea retirada, no integrada o sustituida antes de `dev` actual.

No debe documentarse `AGENT=1`/`OPENCODE=1` como API de entorno vigente.

## Posibles razones para la retirada

**Inferencias, no hechos declarados:**

- `AGENT` es un nombre demasiado genérico y puede colisionar con tooling externo;
- un marker global no distingue primary/subagent ni session identity;
- procesos ejecutados desde una sesión no necesariamente necesitan conocer la abstracción Agent;
- información más precisa podría transportarse mediante contexto explícito si se necesitara.

## `agent-message-previous`

Esta familia, junto con `agent-switch-previous`, intenta preservar causalidad al cambiar selección de agente alrededor de mensajes/eventos.

**Inferencia:** a medida que agent selection se vuelve estado durable de Session, los consumidores necesitan saber qué selección produjo cada transición/mensaje, especialmente para UI, replay y model-follow behavior.

## Agent prompt y step budget

`ConfigAgentV1` define `steps` y el legacy `maxSteps`.

**Hecho confirmado.** El límite pertenece al perfil del agente, no al provider ni a la skill.

Esto refuerza la idea de `Agent.Info` como execution policy:

```text
Agent.Info = prompt + model policy + tool policy + loop budget
```

## Dynamic instructions vs dynamic agents

Conviene separar:

- **dynamic agent resolution:** cargar perfiles desde config;
- **dynamic instructions:** añadir instrucciones de project/workspace/skills/MCP/reference a una ejecución;
- **agent switching:** cambiar qué perfil gobierna la sesión;
- **subagent spawning:** crear otra sesión con otro perfil.

Las branches relacionadas muestran que estos conceptos se han desacoplado progresivamente.

## Conclusiones

- El prompt efectivo de un agente es compuesto y dependiente del entorno.
- Custom/dynamic agents son principalmente perfiles configurables resueltos en runtime.
- References dejaron de justificar un agente separado y pasaron a contexto de `explore`.
- Skills y MCP se anuncian según capacidades/permisos efectivos.
- `AGENT=1` y `OPENCODE=1` existieron como experimento, pero no están en `dev` actual.
- La especialización de OpenCode se concentra en composición de policy/context sobre SessionPrompt, no en runtimes de agente independientes.
