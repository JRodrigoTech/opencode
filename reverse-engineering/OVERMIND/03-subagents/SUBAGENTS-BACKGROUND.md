# Subagents and Background Work — no confundir delegación con clonación

**Status:** CONDITIONAL REFERENCE

## Corrección de enfoque

OpenCode implementa subagents como child sessions detrás de `TaskTool`. Es un patrón sólido **dentro de OpenCode**, pero Overmind no necesita copiarlo para poder usar un agente de coding.

La decisión recomendada para software engineering es:

```text
Overmind Agent
   |
   +-> OpenCode delegation Tool
          |
          v
      OpenCode session
          |
          +-> build / plan / explore / general
          +-> internal child sessions / TaskTool
          +-> background work if OpenCode chooses
          |
          v
      bounded result
```

Overmind trata toda esa jerarquía como una capability externa. No necesita reflejar cada child session de OpenCode en su propio Agent graph.

## Cuándo sí tendría sentido un subagent nativo Overmind

Solo cuando exista una necesidad cognitiva general que pertenezca a Overmind, por ejemplo:

- separar una investigación de Memory/RAG del parent context;
- ejecutar razonamientos con distinto target/model budget;
- delegar una tarea no cubierta por un agente especializado externo;
- correr varias ramas cognitivas propias con ownership explícito.

En ese caso siguen siendo útiles las lecciones de OpenCode:

- identidad separada;
- context isolation;
- explicit grants;
- bounded result;
- depth limits;
- cancellation;
- resume si aporta valor.

Pero son **lecciones de diseño**, no un roadmap obligatorio.

## Abstracción común: solo después

Si en el futuro conviven:

1. OpenCode external agent;
2. uno o más subagents nativos Overmind;
3. quizá otros agent backends;

entonces puede emerger:

```text
AgentDelegationPort
- delegate
- resume
- cancel
- status
```

No diseñarlo antes de tener esos consumidores.

## Background

No implementar `BackgroundJob` porque OpenCode lo tenga. OpenCode puede ejecutar su trabajo interno independientemente.

Overmind necesita background propio solo si una capability de Overmind debe seguir activa más allá de un model turn — watchers, Memory consolidation, MCP connections, scheduler, etc. Entonces debe seguir el contract EVENTS/SERVICES ya definido por Overmind.

## Cognitive economy

Nunca usar el LLM parent para hacer polling del external agent. Si el transporte es síncrono, esperar determinísticamente. Si se vuelve background, un service/event debe detectar completion y despertar cognition solo cuando la policy lo requiera.
