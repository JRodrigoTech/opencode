# 07 — Conclusiones, riesgos y preguntas abiertas

## Modelo mental recomendado

El baseline `dev` se entiende mejor mediante cuatro unidades:

1. **Instance**: scope de proyecto/directorio/workspace y servicios asociados.
2. **Session**: agregado durable y unidad de ejecución/reanudación del agente.
3. **Turn/step loop**: resolución dinámica de agent + model + tools + contexto, seguida de streaming, tool execution y continuación.
4. **Server/API**: host de application services y boundary común para todos los clientes.

## Invariantes confirmados

### 1. La sesión gobierna el runtime

**[CONFIRMADO]** El agente está implementado como un loop stateful alrededor de SessionPrompt/SessionProcessor. Tool calls, compaction, retries y continuaciones pertenecen al lifecycle de sesión.

### 2. La UI está desacoplada del agente

**[CONFIRMADO]** TUI y Desktop consumen la API del backend; no ejecutan directamente SessionPrompt. Incluso la TUI local pasa por `Server.Default().app.fetch` dentro de un Worker.

### 3. Instance es un boundary real

**[CONFIRMADO]** El server headless resuelve instances por request y no requiere un único contexto de proyecto al arrancar. Config y otros servicios son instance-scoped.

### 4. Tools son capabilities reguladas

**[CONFIRMADO]** Registry, exposición efectiva, validación, autorización, ejecución y representación del resultado son fases separadas.

### 5. Provider no equivale a protocol

**[CONFIRMADO]** El package `@opencode-ai/llm` separa adapters de provider y protocolos wire; la capa de sesión aplica policy/context antes de invocarlos.

### 6. La persistencia está migrando hacia SQLite + eventos/proyecciones

**[CONFIRMADO]** SQLite/Drizzle, EventV2 y SessionProjector están activos junto con compatibilidad Session V1 y Storage legacy.

## Tesis evolutiva

**[INFERENCIA]** `dev` está aplicando una migración de tipo strangler en varias dimensiones simultáneas:

```text
monolito packages/opencode
      |
      +--> @opencode-ai/core
      +--> @opencode-ai/server
      +--> @opencode-ai/llm
      +--> @opencode-ai/tui
      +--> @opencode-ai/schema / protocol

Session V1 ------------------> Session V2
legacy events ----bridge-----> EventV2 + projections
legacy/local server ---------> extracted server services
AI SDK-centric path ---------> protocol/native LLM paths
Desktop sidecar V1 ----------> sidecar V2
```

La característica principal de la estrategia es conservar compatibilidad observable mientras los boundaries internos cambian.

## Deuda y riesgos arquitectónicos visibles

### Dualidad V1/V2

**[CONFIRMADO]** Hay servicios, schemas, bridges y rutas de dos generaciones en el mismo composition root.

**Riesgo:** aumenta el número de invariantes que deben mantenerse sincronizadas y dificulta determinar el source of truth de cada entidad sin seguir el flujo concreto.

### Dos abstracciones de persistencia

**[CONFIRMADO]** `Storage` legacy y `Database` SQLite se proveen simultáneamente.

**Riesgo:** migrations, recovery y ownership de datos pueden quedar distribuidos mientras dure la transición.

### Composition root grande

**[CONFIRMADO]** `packages/opencode/src/server/routes/instance/httpapi/server.ts` ensambla un número elevado de services/layers.

**[INFERENCIA]** Aunque los boundaries se han hecho visibles, el wiring todavía centraliza mucho conocimiento de dependencias. Es un lugar excelente para estudiar el dependency graph real.

### Complejidad del provider stack

`provider/provider.ts` y `provider/transform.ts` siguen siendo módulos grandes, coexistiendo con `@opencode-ai/llm` y plugins provider-specific.

**Riesgo:** la frontera entre policy de producto, model metadata y wire protocol puede duplicarse durante la extracción.

### Multiplicidad de runtimes

El sistema ejecuta código en Bun/Node, Worker, Electron y sidecars, y soporta adaptadores específicos por host.

**Riesgo:** lifecycle, signals, filesystem, PTY, proxy y SQLite deben conservar semántica suficiente entre hosts.

### Tool/plugin security surface

**[CONFIRMADO]** Plugins, MCP y `.opencode` pueden ampliar capacidades, mientras tools sensibles acceden a filesystem/procesos.

**Riesgo:** cualquier bypass entre discovery, permission resolution y execution context es de alta severidad. El análisis especializado de tools/permisos debería comprobar precedencia e inheritance exhaustivamente.

## Preguntas abiertas que requieren análisis posterior

### Persistencia/eventos

- ¿Qué operaciones V1 y V2 tienen atomicidad completa entre event append y projection?
- ¿Cómo se recuperan exactamente projections tras un crash a mitad de transición?
- ¿Cuál es el mecanismo de rebuild/replay de eventos durables?

### Runtime del agente

- ¿Qué flags seleccionan exactamente AI SDK vs native LLM runtime en todas las combinaciones?
- ¿Cómo se resuelve la carrera entre abort, retry, tool completion y nuevos inputs?
- ¿Qué estado de `SessionRunState` sobrevive o se reconstruye después de restart?

### Instance/workspace

- ¿Cuál es la clave canónica de identidad de instance frente a workspace/project/worktree?
- ¿Qué caches/services se invalidan al cambiar configuración o filesystem?
- ¿Qué boundaries están preparados para ejecución multi-workspace concurrente en un único daemon?

### UI/backend

- ¿Cuál será el rollout definitivo de `OPENCODE_SIDECAR_V2`?
- ¿Qué estado de TUI/Desktop es puramente derivado y cuál se persiste localmente como UX state?
- ¿Qué garantías de ordering reciben clientes SSE/RPC durante reconnect?

## Prioridades para los análisis especializados

El baseline identifica los seams que deberían explotar los otros agentes del proyecto:

- agents/subagents/skills: `agent/*`, `tool/task.ts`, session parentage;
- prompt/context/compaction: `session/prompt.ts`, `instruction.ts`, `compaction.ts`, prompts model-specific;
- tools/permissions: `tool/*`, `session/tools.ts`, `permission/*`;
- providers: `provider/*`, `session/llm/*`, `packages/llm`;
- session/events: `session/*`, `core/session/*`, `EventV2`, projectors;
- MCP/ACP: protocol adapters y auth/lifecycle;
- backend/transports: server groups/middleware, SSE/WS, sidecars;
- Effect/refactors: layer composition, instance scopes y package extraction.

## Conclusión final

**[INFERENCIA, respaldada por el conjunto del baseline]** OpenCode `dev` ya no debe reverse-engineerearse como una aplicación CLI monolítica. Su arquitectura real es la de un **runtime de agentes session-centric y instance-scoped, alojado detrás de un backend común, con clientes desacoplados y una migración activa hacia servicios Effect, eventos durables, SQLite y packages/protocol adapters especializados**.

La coexistencia de implementaciones antiguas y nuevas no es ruido: es una de las fuentes de evidencia más valiosas para descubrir los boundaries que el propio equipo considera suficientemente estables como para extraerlos.