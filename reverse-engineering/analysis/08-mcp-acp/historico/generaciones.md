# Generaciones de diseño MCP/ACP

## Resumen

La historia observada no es una secuencia lineal de branches donde cada nombre sustituye al anterior. Hay dos líneas principales con ritmos distintos:

1. **MCP** evoluciona desde un cliente integrado en core hacia un subsistema de integración stateful, con lifecycle, OAuth y contratos públicos.
2. **ACP** nace más tarde como una anti-corruption layer que proyecta el runtime persistente de OpenCode hacia un protocolo de clientes de agentes.

## Generación MCP 0: integración temprana dentro de core

### Evidencia

Branches históricas como `mcp-core-skeleton`, `mcp-adjustment-work` y la familia basada en `v2` sitúan MCP dentro de `packages/core/src/mcp/`. `mcp-prompts` contiene un merge explícito de `origin/v2` y conflicto sobre `packages/core/src/mcp/index.ts`.

### Hechos confirmados

- MCP estaba más concentrado en una unidad de core.
- tool/prompt/resource discovery aparecía más cerca del momento de conexión.
- la línea `v2` incorporaba administración CLI de servidores y flows de integración/auth.

### Inferencia

El propósito era demostrar que OpenCode podía consumir MCP de extremo a extremo antes de fijar boundaries estables. El coste era una mayor mezcla entre transport, discovery, auth, configuración y consumo.

## Generación MCP 1: lifecycle y separación de concerns

### Evidencia en `dev`

El subsistema termina separado en:

- `mcp/index.ts` — lifecycle y state;
- `mcp/catalog.ts` — discovery/adaptation;
- `mcp/oauth-provider.ts` — protocolo OAuth MCP;
- `mcp/auth.ts` — persistencia auth;
- routes/API — exposición a clientes OpenCode.

### Cambios de diseño

#### De resultado implícito a state machine explícita

`refactor/mcp-connection-results@2a25bdce` formaliza resultados `connected/unavailable` y mantiene status de dominio como `needs_auth` o `needs_client_registration`.

#### De un transport remoto esperado a negociación de compatibilidad

El runtime vigente intenta Streamable HTTP y después SSE, conservando stdio para local.

#### De cleanup accidental a lifecycle scoped

Clients/transports pasan a ser recursos cerrables dentro del scope Effect.

### Inferencia

Esta generación convierte MCP en un servicio autónomo y hace posible que TUI, ACP, server y SDK consuman una misma instancia sin replicar conexión/protocolo.

## Generación MCP 2: autenticación robusta y boundary no confiable

### Evidencia

- OAuth provider persistente.
- refresh de tokens antes de conexión.
- `ecf550c8`: cancelación real de refresh bloqueado.
- `4d4eb7de`: callback loopback sin puertos fijos.
- `restrict-mcp-env@8f626456`: entorno mínimo para subprocess locales.
- `kit/mcp-tolerate-bad-output-schemas`: tolerancia a servers con schemas imperfectos.

### Hechos confirmados

OpenCode protege su lifecycle frente a:

- I/O OAuth que no termina;
- servers sin dynamic registration;
- datos/schemas defectuosos;
- filtración indiscriminada de variables del proceso padre;
- divergencias de transport entre generaciones MCP.

### Inferencia

MCP deja de considerarse una extensión confiable instalada localmente y pasa a tratarse como boundary de integración externo con fallos parciales esperables.

## Generación MCP 3: resources/eventos/API pública

### Evidencia

- `feat/mcp-resource-list-changed@c03c2be4`.
- `e1d1352d`: resource catalog/read expuestos por API/SDK.
- `4d3ff368`: refresh frontend simplificado.

### Cambio de diseño

Resources ya no son solo material potencial para construir prompts. Se vuelven una capacidad consultable por clientes OpenCode con identidad `{server, uri}` y eventos de invalidación.

### Hechos confirmados

- soporte a resources y resource templates;
- `readResource`;
- evento `mcp.resources.changed`;
- capability check `resources.listChanged`;
- protección frente a notificaciones de clients MCP obsoletos.

### Inferencia

Este es el punto en que OpenCode pasa de “usar MCP” a **ofrecer MCP como parte de su plataforma interna**, encapsulando el protocolo tras su propia API.

## Qué ideas MCP llegaron a `dev`

### Confirmado

- stdio local;
- Streamable HTTP + SSE fallback remoto;
- status semántico de conexión;
- OAuth y dynamic client registration;
- refresh de tokens;
- callback loopback;
- tool discovery y namespaces;
- tool list change notifications;
- prompts;
- resources/templates;
- resource list change notifications;
- APIs de resources;
- lifecycle scoped;
- tolerancia a input/output schemas imperfectos;
- ambiente restringido para procesos locales.

## Qué ideas MCP cambiaron

### Discovery eager generalizado

**Hecho confirmado.** La generación temprana estaba más orientada a discovery conjunto tras conectar. En la arquitectura vigente, las tools mantienen cache/invalidation porque forman parte del runtime ejecutable; prompts/resources se obtienen de manera más lazy.

**Inferencia.** El cambio reduce startup cost y desacopla la salud de capabilities secundarias.

### Monolito MCP en core

**Hecho confirmado.** El código termina dividido por lifecycle, catálogo y auth.

**Inferencia.** La separación responde a que MCP se volvió dependencia transversal de varios clientes y ya no podía permanecer como helper privado del agente.

## Generación ACP 0: `nxl-acp-v1`

### Evidencia

`nxl-acp-v1@bb84f713` contiene un commit propio respecto a un merge-base histórico común con la línea `nxl-acp-lifecycle`.

### Inferencia soportada

Representa el primer adapter viable: Agent ACP + Service sobre SDK OpenCode. El objetivo principal era mapear lifecycle básico de sesión/prompt sin reemplazar el runtime.

## Generación ACP 1: lifecycle/event-driven adapter

### Evidencia

`nxl-acp-lifecycle@26d3f2f1` añade un segundo commit sobre la misma familia y toca Agent, Event, Permission, Service y tests.

La implementación consolidada en `dev` usa:

- stream global de eventos;
- replay para sesiones cargadas;
- espera hasta `idle`;
- traducción de tool states;
- permissions bridge;
- cancelación de turns;
- load/resume/fork.

### Cambio de diseño

ACP deja de ser una simple colección de RPCs y pasa a ser una stateful projection coordinada con la máquina de estados de Session.

### Inferencia

Esta generación descubre el boundary real: una request ACP no puede completarse correctamente observando solo la respuesta HTTP del submit; necesita escuchar la ejecución asíncrona de OpenCode.

## Generación ACP 2: configuración y reconstrucción durable

### Evidencia

La existencia de `acp-config-commit` y el `ACPService` vigente muestran una capa dedicada a:

- catálogos por cwd;
- configOptions;
- model/mode/variant;
- recuperación de configuración desde historial;
- registro de MCP servers aportados por el cliente ACP.

### Inferencia

El adapter evoluciona desde “controlar un turn” a “reconstruir un workspace/session environment” compatible con un IDE/cliente de larga duración.

## Generación ACP 3a: elicitation experimental

### Evidencia

`a36c8392` (`feat(cli): support acp elicitation`) añade:

- capability negotiation;
- `unstable_createElicitation`;
- mapping de OpenCode Forms a schema ACP;
- respuesta ACP de vuelta a `form.reply`;
- cancelación cuando la forma no es representable.

### Estado frente a `dev`

**Hecho confirmado.** El símbolo/bridge equivalente no aparece en la baseline vigente inspeccionada.

### Clasificación

**Experimental / no consolidado.** No debe presentarse como capacidad actual.

### Inferencia

El experimento demuestra el camino de diseño para futuras interacciones estructuradas: reutilizar el dominio Form interno y adaptar únicamente cuando el cliente ACP anuncia una representación compatible.

## Generación ACP 3b: subagent projection experimental

### Evidencia

`acp-subagent-events@ca357ee2` añade:

- child-session registry;
- cálculo de root/depth;
- proyección de eventos child a sesión ACP root;
- `_meta["opencode/child-session"]`;
- toolCall IDs namespaced por child;
- permission context para child sessions.

### Estado

Se documenta como evolución específica, no como parte asumida de `dev`.

### Inferencia

La branch resuelve una incompatibilidad de modelos: OpenCode usa sesiones hijas reales para subagentes; ACP puede necesitar mantener una única sesión lógica ante el cliente. La metadata permite transportar jerarquía sin alterar el lifecycle ACP base.

## Relación MCP ↔ ACP

### Hecho confirmado

ACP acepta definiciones de MCP servers y las registra en OpenCode. Después:

1. OpenCode MCP Service establece conexión/auth/discovery.
2. Las MCP tools entran en el catálogo de herramientas del runtime.
3. Session ejecuta tools de forma normal.
4. ACP observa el `ToolPart` resultante y lo proyecta como tool call/update.

### Consecuencia arquitectónica

**Inferencia fuerte.** MCP y ACP están en lados opuestos del runtime:

- MCP conecta **OpenCode → proveedores de capacidades/tools/resources**.
- ACP conecta **clientes/IDEs → OpenCode como agente**.

No existe un puente directo “ACP tool call → MCP call” en el diseño observado. La sesión y el tool runtime de OpenCode son el punto de composición.

## Diagrama evolutivo

```text
MCP
core/v2 skeleton
    │
    ├─ prompts / CLI / auth experiments
    ▼
stateful MCP service
    ├─ transport fallback
    ├─ OAuth lifecycle
    ├─ tool cache + notifications
    ├─ catalog abstraction
    ▼
platform capability
    ├─ resources/templates API
    ├─ resource events
    └─ SDK/TUI consumption

ACP
nxl-acp-v1
    ▼
nxl-acp-lifecycle
    ├─ event stream
    ├─ permissions
    ├─ durable sessions
    ▼
config/session reconstruction
    ├─ models/modes/variants
    └─ MCP server registration
    ├───────────────┬────────────────┐
    ▼               ▼                │
elicitation      subagent events     │
(experimental)   (evolution branch)  │
```

## Ideas descartadas, reformuladas o no consolidadas

### Confirmadas como no presentes en la baseline observada

- bridge ACP `unstable_createElicitation` del commit `a36c8392`.

### Reformuladas

- discovery MCP eager → tools cacheadas + prompts/resources lazy.
- resultados de conexión por shape implícito → tagged domain results.
- resources internos → API/SDK pública con eventos.

### No atribuibles con seguridad

- branches cuyo nombre sobrevive pero HEAD actual ya representa refactors posteriores (`mcp-attachments`).
- refs devueltos por el índice pero no resolubles (`acp-pager`).

## Boundaries reales revelados por la evolución

### MCP boundary

Ownership de:

- transport;
- auth;
- capabilities externas;
- discovery;
- normalización;
- naming;
- refresh/invalidation.

No ownership de:

- session lifecycle;
- tool permission policy;
- agent runtime.

### ACP boundary

Ownership de:

- wire protocol ACP;
- lifecycle de conexión ACP;
- mapping session/config/content/events;
- proyección de permissions/tool updates.

No ownership de:

- persistencia autoritativa;
- ejecución de tools;
- provider/model runtime;
- MCP transport/auth.

## Conclusión

La evolución de ambos protocolos refuerza una arquitectura en capas: OpenCode se sitúa como runtime autoritativo en el centro, MCP aporta capacidades hacia dentro y ACP expone el agente hacia fuera. Las branches históricas más importantes no son las que más código contienen, sino las que hacen explícitos esos boundaries: `nxl-acp-lifecycle`, `refactor/mcp-connection-results`, la evolución OAuth, resources/events y los experimentos de elicitation/subagent projection.