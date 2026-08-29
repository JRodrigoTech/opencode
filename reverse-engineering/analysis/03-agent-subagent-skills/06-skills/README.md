# Skills: discovery, activación y evolución

## Qué es una skill en OpenCode

Una skill es un paquete de instrucciones especializadas, normalmente representado por un `SKILL.md` con metadata y contenido asociado.

No es un agente y no tiene loop propio.

```text
Agent = cómo se ejecuta una sesión
Skill = contexto/workflow especializado que puede cargarse en esa ejecución
```

## Diseño vigente en `dev`: lazy activation

La arquitectura actual separa deliberadamente **discovery** y **activation**.

### Discovery

`packages/opencode/src/skill/index.ts` y `packages/opencode/src/skill/discovery.ts` construyen un catálogo de skills desde varias fuentes.

Fuentes observadas en la evolución vigente:

- directorios `.opencode/skill` / `.opencode/skills` compatibles según generación;
- `.claude/skills`;
- `.agents/skills`;
- paths adicionales de configuración;
- fuentes remotas descargadas/cacheadas.

`read-global-claude-skills` confirma discovery global desde `~/.claude/skills`.

`agents-skill-compat` confirma que la compatibilidad con `.claude` y `.agents` se trata específicamente como **skill sources**, evitando que esos directorios externos contaminen indiscriminadamente otras clases de configuración.

### Catálogo en system prompt

`packages/opencode/src/session/system.ts` ejecuta `SystemPrompt.skills(agent)`.

Si la tool `skill` está deshabilitada por permisos, no se anuncia el catálogo.

Si está habilitada:

1. obtiene `skill.available(agent)`;
2. inserta una explicación de que las skills contienen workflows especializados;
3. muestra un catálogo formateado con nombre/descripción;
4. instruye al modelo para usar la tool `skill` cuando una tarea coincida.

**Hecho confirmado.** El cuerpo completo de todos los `SKILL.md` no se inserta preventivamente en cada prompt.

### Activation

`packages/opencode/src/tool/skill.ts` recibe el nombre de una skill y:

- resuelve la skill disponible;
- evalúa/solicita permiso `skill:<name>`;
- devuelve las instrucciones completas y referencias/resources de esa skill.

`packages/opencode/src/tool/skill.txt` describe explícitamente esta operación como “inject the skill's instructions and resources into current conversation”.

### Consecuencia

```text
Discovery time:
  name + description + location
          │
          ▼
System prompt catalog
          │
      model decides
          │
          ▼
skill(name)
          │ permission
          ▼
full SKILL.md + resources
```

Esto reduce consumo de contexto y hace permission-aware la carga de conocimiento especializado.

## Primera generación: `feature/agent-skills`

Esta familia representa la introducción de skills como extensión del sistema de agentes.

**Lectura evolutiva confirmada por el estado posterior:** inicialmente agents y skills estaban más próximos conceptualmente; la arquitectura actual los desacopla y hace que cualquier agente autorizado pueda consultar el catálogo.

## `feature/skill-tool`

Esta rama representa la transición clave hacia una tool explícita de activación.

El diseño que termina en `dev` es precisamente:

```text
system prompt advertises skill
          +
SkillTool loads it lazily
```

**Inferencia con evidencia convergente:** el objetivo fue evitar inyectar todas las instrucciones de skills en el prompt base y dejar al LLM elegir cuándo pagarlas en contexto.

## `migrate-skill-discovery`

La implementación inspeccionada de esta línea descubre `skills/**/SKILL.md` en directorios externos `.claude` y `.agents`, además de fuentes OpenCode/configuradas.

También mantiene información de directorios y deduplicación/precedencia entre fuentes.

**Hecho confirmado.** La discovery se fue moviendo hacia un servicio propio en lugar de ser una extensión ad hoc de prompt loading.

## `read-global-claude-skills`

Commit representativo `d3b820d0ae1ffe909e9e82bd6213b964b73ffe5e` ajusta tests para crear explícitamente una skill en `~/.claude/skills` y verificar que se descubre globalmente.

Esto confirma compatibilidad deliberada con el ecosistema Claude.

## `agents-skill-compat`

Commit `a5fed48b3941ab2aef2683f3313713237ca24276` — `refactor(core): scope compatibility sources to skills`.

El cambio evita incluir siempre `.claude`/`.agents` como entries genéricos de config y, en su lugar, los consulta desde el plugin de skills.

**Interpretación:**

```text
.claude / .agents
      │
      └─ compatibility source for skills

NO

.claude / .agents
      └─ generic configuration root for everything
```

Es un tightening de boundary importante.

## Remote skill discovery

`packages/opencode/src/skill/discovery.ts` contiene lógica para descargar/persistir fuentes remotas en cache antes de incorporarlas al catálogo.

**Hecho confirmado.** Discovery remoto y activation son fases distintas: descargar/indexar no equivale a insertar automáticamente la skill en el contexto del modelo.

## Mentions e invocación explícita

### `fix-skill-mentions`

La línea reciente separa operaciones como `Skill.resolve` y `Skill.prepare`.

**Lectura:** resolver identidad/source y preparar contenido para la sesión son responsabilidades distintas.

### `skill-mention-bypass`

Una skill solicitada explícitamente por el usuario puede evitar una aprobación redundante, pero la branch restringe ese bypass al contexto concreto de la tool/user request.

**Principio confirmado por el cambio:** explicit user intent no debe convertirse en un override global del permission layer.

## `dev-multi-skills`

Commit `5e5e3d09cbb4c321032416887255d1067fefadae` — `feat(tui): support multiple inline skills`.

La branch permite componer varias skills en una misma invocación de command:

```text
/skill-alpha /skill-beta inspect src/foo.ts
```

El runtime:

- detecta commands cuyo source es `skill`;
- encuentra skills adicionales en los argumentos;
- elimina duplicados;
- concatena templates de las skills seleccionadas;
- conserva argumentos restantes.

**No asumir como baseline universal.** Es una línea específica de composición inline de skills y no cambia la arquitectura fundamental de discovery + activation.

## Segunda generación: skills como estado/evento de Session (`core/v2`)

Las ramas de agosto de 2026 muestran una arquitectura alternativa más ambiciosa.

### `skill-source-observer`

Extrae observación/watch de fuentes a un componente específico `SkillSourceObserver`.

**Interpretación:** discovery pasa de “scan when needed” hacia una fuente reactiva observable.

### `session-skill-activation`

La activación se vincula a `Session` y publica un evento durable del tipo:

```text
SessionEvent.Skill.Activated {
  id,
  name,
  text
}
```

La sesión puede reanudarse después de la activación.

**Hecho confirmado en esa branch.** La activación deja de ser sólo output de una tool y se convierte en estado/evento explícito de la conversación.

### Diferencia con `dev`

`dev`:

```text
SkillTool output becomes conversation context
```

`core/v2` experimental:

```text
Skill activation becomes a first-class Session event
```

La segunda opción mejora replay/synchronization/observability, pero no debe describirse como comportamiento vigente de `dev`.

## IDs, names y compatibilidad

`agent-settings-skew` termina en un fix de app para skills sin `id`: usa `skill.id ?? skill.name` al construir suggestions/mentions.

Esto revela una diferencia entre fuentes de skill: no todas garantizan exactamente la misma identidad estructural.

**Inferencia:** la capa de compatibility debe normalizar una identidad estable si activation durable/event sourcing se convierte en arquitectura principal.

## `effectify-skill`

El tip de la branch inspeccionado sólo modifica lockfile/dependencias. Por tanto:

- su nombre sugiere una migración a Effect;
- **no se atribuye una implementación concreta al tip** sin seleccionar commits históricos anteriores.

Se clasifica como rama de refactor potencial, no como evidencia primaria.

## `kit/skill-lazy-init`, `kit/retrofit-skill-main`, `styled-skill-errors`

Por nomenclatura son relevantes al subsistema, pero no se usan como evidencia de arquitectura central aquí sin inspección de commits específicos.

Se conservan en el inventario para análisis posterior si se necesita granularidad de implementación/error handling.

## Boundaries reconstruidos

```text
Skill Source
   │
   ▼
Discovery / Observer
   │
   ▼
Normalized Skill metadata
   │
   ├─► SystemPrompt catalog
   │
   └─► SkillTool / Session activation
            │
            ├─ permission
            └─ full content/resources
```

## Conclusiones

- Skills no son subagents.
- `dev` implementa carga lazy: catálogo ligero primero, instrucciones completas después.
- Discovery es multi-source y compatible con `.claude`/`.agents`.
- Activation está permission-gated.
- La línea `core/v2` intenta promover skill activation a evento durable de Session.
- La evolución refuerza un boundary claro entre source discovery, identity resolution, permission y session activation.
