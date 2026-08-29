# Inventario y análisis de branches

Este documento registra la primera fase de ingeniería inversa del árbol de branches del repositorio `JRodrigoTech/opencode`.

> Baseline de comparación: `dev`, que es el **default branch** actual del repositorio y, para este estudio, se considera el main branch funcional.

## Resumen ejecutivo

- Branches remotas detectadas: **1.595**.
- Default branch: `dev`.
- Tip de `dev` observado durante el análisis: `dc4449df0d52199704ea4989a5a993ebbc605612`.
- `production` está contenido en `dev` y se encontraba **2 commits por detrás**, sin commits exclusivos.
- `2.0` está completamente contenido en `dev` y se encontraba **4.481 commits por detrás**, por lo que representa una línea histórica ya absorbida.
- `beta` y `v2` forman una línea histórica/experimental fuertemente divergida respecto a `dev`: ambas aparecían con **2.627 commits exclusivos** y **1.098 commits exclusivos de `dev`** desde el mismo merge-base `0e2dd4ad150d0182fc9e43d81424d8db11465977`.
- `core` es una branch histórica de extracción/refactor de utilidades compartidas hacia `packages/core`, con 3 commits exclusivos respecto a su merge-base y una divergencia muy grande con el `dev` actual.
- `sdk` es una branch histórica de evolución del SDK/OpenAPI que introducía una superficie `v2` generada y cambios de consumidores internos.

La enorme cantidad de branches indica que este repositorio conserva un volumen significativo de ramas efímeras de features, fixes, spikes, reproducciones, migraciones y experimentos. Por tanto, para reverse engineering no conviene asumir que la existencia de una branch implica que su diseño siga vigente.

## Metodología

Para reconstruir la función de una branch se consideran, por este orden:

1. Relación genealógica contra `dev`: ahead/behind y merge-base.
2. Commits exclusivos de la branch.
3. Paths modificados desde el merge-base.
4. Pull requests o referencias históricas cuando existen.
5. Naming de la branch únicamente como evidencia secundaria.

### Niveles de evidencia

- **Alta**: función confirmada por commits exclusivos y diff contra `dev`/merge-base.
- **Media-alta**: branch ya absorbida por `dev`; función reconstruida mediante tip histórico, genealogía y paths conocidos.
- **Media**: naming muy descriptivo y consistente con familias de trabajo del repositorio, pero sin suficientes commits exclusivos actuales para confirmar el detalle.
- **Baja**: nombre opaco, branch de prueba o branch generada automáticamente sin contexto suficiente.

## Tabla de branches estructurales y de larga vida

| Branch | Estado frente a `dev` | Función reconstruida | Evidencia | Detalle |
|---|---|---|---|---|
| `dev` | baseline | Main/default branch de desarrollo | Alta | [ver](#branch-dev) |
| `production` | behind 2 / ahead 0 | Snapshot de producción muy próximo a `dev` | Alta | [ver](#branch-production) |
| `beta` | diverged: ahead 2627 / behind 1098 | Línea beta/prerelease e integración de cambios de nueva arquitectura | Alta | [ver](#branch-beta) |
| `v2` | diverged: ahead 2627 / behind 1098 | Línea histórica de arquitectura v2; comparte genealogía actual con `beta` | Alta | [ver](#branch-v2) |
| `2.0` | behind 4481 / ahead 0 | Punto histórico de la transición 2.0, ya absorbido por `dev` | Alta | [ver](#branch-20) |
| `core` | diverged: ahead 3 / behind 3986 | Extracción de utilidades y runtime común hacia `packages/core` | Alta | [ver](#branch-core) |
| `sdk` | diverged: ahead 6 / behind 10686 | Evolución del SDK/OpenAPI con cliente `v2` generado | Alta | [ver](#branch-sdk) |
| `reverse-engineering` | fuera del upstream funcional | Documentación de este estudio | Alta | [ver](#branch-reverse-engineering) |

<a id="branch-dev"></a>
## `dev`

`dev` es el default branch del repositorio y el baseline de todo el análisis. El tip observado fue `dc4449df0d52199704ea4989a5a993ebbc605612`, cuyo commit era `chore: update nix node_modules hashes`.

Para la ingeniería inversa, cualquier branch se interpreta en relación con `dev`: una branch completamente detrás se considera histórica/absorbida; una branch por delante contiene trabajo no integrado; una branch divergida contiene una línea de diseño alternativa o un desarrollo prolongado que se separó del mainline.

**Nivel de evidencia:** alta.

<a id="branch-production"></a>
## `production`

`production` se encontraba exactamente 2 commits por detrás de `dev`, con `ahead=0`. Esto implica que no contiene una implementación alternativa actual: es esencialmente un snapshot de release/deployment extremadamente cercano al mainline.

Su utilidad para reverse engineering es limitada como fuente de features exclusivas, pero resulta útil para identificar qué commits pueden quedar fuera temporalmente del despliegue productivo.

**Nivel de evidencia:** alta.

<a id="branch-beta"></a>
## `beta`

`beta` es una línea de integración/prerelease considerablemente separada del `dev` actual. La comparación observada fue `ahead=2627`, `behind=1098`, con merge-base `0e2dd4ad150d0182fc9e43d81424d8db11465977`.

El tip observado era `106629aa118086be7def6123241a9bf056ba77b6` con el commit `feat(infra): deploy beta web app with SST (#46086)`. El diff contra `dev` mostraba, entre otros cambios, una reestructuración fuerte de infraestructura, workflows y una nueva capa `packages/ai` con protocolos y providers nativos para Anthropic, Bedrock, Gemini, OpenAI/Open Responses, Azure, xAI, Z.AI, OpenRouter, Cloudflare y otros.

Esto convierte `beta` en una fuente especialmente relevante para reverse engineering porque expone decisiones arquitectónicas que no coinciden todavía con `dev`: abstracción AI propia, protocolos, routing, cache policy, imágenes, WebSocket/continuations y una expansión fuerte de tests mediante recordings.

**Nivel de evidencia:** alta.

<a id="branch-v2"></a>
## `v2`

La comparación de `v2` contra `dev` produjo exactamente la misma relación principal observada en `beta`: `ahead=2627`, `behind=1098` y el mismo merge-base `0e2dd4ad150d0182fc9e43d81424d8db11465977`.

El conjunto de archivos devuelto por la comparación coincide con la gran línea de nueva arquitectura observada en `beta`, incluyendo la introducción/expansión de `packages/ai`, cambios importantes de infraestructura y reorganización del stack de proveedores.

A efectos de ingeniería inversa, `v2` debe tratarse como una rama histórica/arquitectónica principal y compararse especialmente con `dev` para identificar qué ideas de la transición v2 fueron adoptadas, reformuladas o abandonadas.

**Nivel de evidencia:** alta.

<a id="branch-20"></a>
## `2.0`

`2.0` estaba `ahead=0` y `behind=4481` frente a `dev`. Su propio tip era el merge-base de la comparación, por lo que su contenido está completamente en la historia de `dev`.

No representa trabajo exclusivo actual; representa un checkpoint histórico de la evolución 2.0. Su valor es cronológico: permite estudiar cómo era el sistema antes de miles de commits posteriores y comparar la arquitectura temprana con la implementación moderna.

**Nivel de evidencia:** alta para la genealogía; media-alta para su propósito histórico exacto.

<a id="branch-core"></a>
## `core`

`core` estaba divergida respecto a `dev`: `ahead=3`, `behind=3986`, merge-base `e4be55792861504f23d055a37767357ee40b1d83`.

La comparación muestra la creación de `packages/core` con piezas como:

- runtime y observability basados en Effect;
- filesystem abstraction;
- helpers globales;
- manejo de NPM;
- utilidades de locking/flock;
- glob, path, retry, hash, slug, identifier y otras primitivas;
- una batería extensa de tests para filesystem y flock.

La intención parece haber sido extraer primitivas infraestructurales reutilizables desde el paquete principal hacia un core independiente. Aunque esta branch quedó muy atrás del mainline actual, es útil para entender intentos de modularización del código base.

**Nivel de evidencia:** alta.

<a id="branch-sdk"></a>
## `sdk`

`sdk` estaba divergida con `ahead=6` y `behind=10686`, merge-base `6667856ba5dac3e5dd77c7008cee2d09be894472`.

La superficie del diff revela una evolución importante de `packages/sdk`, especialmente:

- expansión de `openapi.json`;
- generación de un cliente `packages/sdk/js/src/v2`;
- serializers de paths/query/body;
- SSE support;
- tipos y SDK generado de gran tamaño;
- cambios en server/client helpers;
- adaptación de consumidores del SDK en desktop, TUI, ACP y server.

Es una branch histórica valiosa para estudiar cómo se intentó estabilizar una API v2 y reducir el acoplamiento de consumidores internos a detalles del backend.

**Nivel de evidencia:** alta.

<a id="branch-reverse-engineering"></a>
## `reverse-engineering`

Branch creada específicamente para almacenar esta documentación. No representa una feature upstream de OpenCode y debe excluirse de cualquier inferencia sobre la arquitectura original del agente.

**Nivel de evidencia:** alta.

## Familias de branches observadas

El inventario completo de 1.595 ramas contiene familias claramente reconocibles. Estas familias son especialmente útiles para organizar las siguientes fases de investigación.

### Agente, subagentes y skills

Ejemplos: `add-dynamic-agents-resolving`, `agent-env-markers`, `agent-model-follow`, `agent-permission-order`, `agent-picker-custom`, `agent-picker-variants`, `agents-skill-compat`, `dev-multi-skills`, `effectify-skill`, `feature/agent-skills`, `feature/skill-tool`, `feat/reference-agent`, `implement-background-agents`, `nxl/background-subagents`, `nxl/small-model-subagents`, `read-global-claude-skills`, `session-skill-activation`, `skill-source-observer`, `subagent-command`, `subagent-command-v2`, `subagent-model-routing`, `subagent-observability`, `subagent-permission-inheritance`, `subagent-resume`.

**Interés RE:** muy alto. Esta familia permite reconstruir agent selection, subagent lifecycle, inheritance, model routing, skills discovery y mecanismos de background execution.

### Prompt, context y compaction

Ejemplos: `adjust-instructions-logic`, `bounded-compaction`, `cache-friendly-compaction`, `compaction-adjustments`, `compaction-cleanup`, `compaction-model-marker`, `compaction-steer`, `compaction-study`, `context-checkpoint`, `context-overflow`, `context-weight`, `instruction-read-race`, `namespace-instructions`, `openai-compaction`, `prompt-attachments`, `prompt-retry`, `refactor/session-prompt-parts`, `system-context`.

**Interés RE:** crítico. Aquí se puede reconstruir cómo OpenCode arma el contexto, carga instrucciones, compone prompts, maneja overflow y decide cuándo compactar.

### Tool execution, permissions y code mode

Ejemplos: `apply-patch`, `ask-question-tool`, `canonical-tool-names`, `code-mode-boundary`, `code-mode-opencode`, `codemode-opencode-adapter`, `codemode-service`, `deferred-tools`, `execute-code-mode-v2`, `permission-alias`, `permission-panel`, `permission-rework`, `pinned-tools`, `raw-tool-recovery`, `simulated-tools`, `tool-input-repair`, `tool-result-api`, `tool-transform-api`, `truncate-to-file`.

**Interés RE:** crítico. Esta familia expone el registry de tools, schemas, policy/permission checks, output truncation, patching y recuperación de errores.

### Providers, LLM protocols y model routing

Ejemplos: `ai-native-routing`, `anthropic-fixes`, `anthropic-thinking-variants`, `azure-chat-reasoning`, `bedrock-credential-chain`, `cloudflare-prompts`, `codex-auth`, `deepseek-responses`, `gemini-thinking-level`, `github-copilot-parity`, `llm-centralization`, `llm-native-event-adapter`, `native-provider-core`, `openai-compatible-native-ai`, `openai-websocket`, `provider-config-merge`, `provider-credentials`, `reasoning-options`, `responses-api-adjustments`, `vertex-fixes`, `xai-oauth`.

**Interés RE:** crítico. Permite reconstruir la capa de abstracción de providers, compatibilidad entre protocolos, reasoning, continuations, auth y selección de modelos.

### MCP / ACP

Ejemplos: `acp-config-commit`, `acp-subagent-events`, `mcp-attachments`, `mcp-config-events`, `mcp-core-skeleton`, `mcp-oauth-refresh`, `mcp-prompts`, `mcp-resource-api`, `mcp-resource-content`, `mcp-resource-ui`, `mcp-search-tool`, `nxl-acp-elicitation`, `nxl-acp-lifecycle`, `nxl-acp-v1`.

**Interés RE:** muy alto. Permite entender protocolos de integración, recursos, prompts, auth, elicitation, lifecycle y propagación de eventos.

### Sessions, events y persistence

Ejemplos: `archive-session`, `explicit-session-selection`, `explicit-turn-api`, `message-v3`, `orphan-session-recovery`, `protocol-events`, `published-events`, `session-archival`, `session-event-stream`, `session-forking`, `session-history-api`, `session-list-index`, `session-model-sync`, `session-prompt-history`, `session-store-reads`, `storage-v2-service`.

**Interés RE:** crítico. Aquí reside buena parte del state machine del agente: sesiones, mensajes, turns, eventos, persistencia, replay y recovery.

### Server, transport y daemon/service

Ejemplos: `app-backend-v2`, `backend-adapter-v2`, `daemon-election`, `embedded-server-dispose`, `native-http-middleware`, `reconnect-backoff`, `server-discovery`, `server-location-paths`, `service-election-safety`, `service-restart`, `service-shutdown-watchdog`, `websocket-auth`, `websocket-rpc`, `websocket-upgrade-diagnostics`.

**Interés RE:** muy alto. Permite reconstruir procesos residentes, daemon election, discovery, transports, reconnect y lifecycle del backend.

### TUI, desktop y browser UI

Ejemplos: `activity-panel`, `app/startup-splash`, `browser-app-ui`, `desktop-browser`, `desktop-v2-daemon`, `desktop-v2-sidecar`, `tui-child-picker`, `tui-claude-style`, `tui-experimental-design`, `tui-inbox-tabs`, `tui-plugin-load`, `tui-prompt-state`, `timeline-scroll-state`, `tabs-restore`.

**Interés RE:** alto, especialmente para reconstruir cómo se reflejan internamente los estados del agente y qué eventos/SDK consume cada interfaz.

### Effect migration / service architecture

Ejemplos: numerosas branches `effect/*`, `kit/effect-*`, `effect-auth-foundation`, `effect-drizzle-sqlite`, `effect-dynamic-tools`, `effect-plugin-adapter`, `refactor/effect-pattern-migration`.

**Interés RE:** muy alto. Refleja una migración progresiva a Effect, services/layers, typed errors y runtime composition.

### HTTP API y SDK

Ejemplos: `add-api-shape`, `api-dsl-cleanup`, `direct-sdk`, `explicit-turn-api`, múltiples `kit/httpapi-*`, `openapi-names`, `operation-identifiers`, `question-endpoint`, `sdk-consumer-types`, `sdk-generation-dev-check`.

**Interés RE:** crítico para reconstruir el contrato externo del agente y cómo se mapea a operaciones internas.

### Workspaces, worktrees, filesystem y VCS

Ejemplos: `core-project-relocation`, `deferred-session-move`, `git-head-watch`, `jj-vcs-plugin`, `project-copy-migration`, `remote-workspaces-plan`, `snapshot-repository`, `vcs-branch-metadata`, `workspace-copy`, `workspace-plan`, numerosas `worktree-*` y `layer-node-*`.

**Interés RE:** muy alto. Permite reconstruir project detection, cwd/location model, repository snapshots, worktrees y VCS abstraction.

### Reproducciones, debugging y audits

El árbol contiene muchas branches `repro/*`, `debug-*`, `electron-audit-*`, `architecture-review`, `codemode-audit`, `write-audit`, `worktree-audit-effect-services`, además de branches generadas por `claude/*`, `copilot/*` y `opencode/*`.

**Interés RE:** alto como fuente de invariants y failure modes. Una branch de reproducción suele revelar con mucha precisión qué contrato interno se esperaba que no se rompiera.

## Observaciones sobre el inventario completo

El listado recuperado contiene **1.595 nombres de branch** y cubre, entre otras, las siguientes series/prefijos: `fix/*`, `feat/*`, `feature/*`, `refactor/*`, `cleanup/*`, `chore/*`, `test/*`, `effect/*`, `kit/*`, `nxl/*`, `claude/*`, `copilot/*`, `opencode/*`, `worktree-*`, `layer-node-*`, `mcp-*`, `llm-*`, `provider-*`, `session-*`, `shell-*`, `tool-*`, `tui-*`, `desktop-*`, `browser-*`, `sdk-*`, `v2-*`, `oc-*` y múltiples ramas sin prefijo.

No debe confundirse este inventario con 1.595 arquitecturas distintas. Una gran parte son ramas efímeras, spikes, reproducciones o fixes ya absorbidos. Para ingeniería inversa, la prioridad debe darse a las ramas que introducen conceptos nuevos o muestran estados intermedios de refactors mayores.

## Priorización para las siguientes fases

Orden recomendado de análisis detallado:

1. `dev` actual: arquitectura real vigente.
2. `beta` / `v2`: arquitectura alternativa o futura y diferencias con `dev`.
3. Familias `agent-*`, `subagent-*`, `skill-*`: comportamiento del agente.
4. `prompt-*`, `context-*`, `compaction-*`, `instruction-*`: construcción de contexto.
5. `tool-*`, `permission-*`, `code-mode-*`: tool runtime y policy.
6. `provider-*`, `llm-*`, `native-provider-*`, `responses-*`: comunicación con modelos.
7. `session-*`, `message-*`, `protocol-events`: state machine y persistence.
8. `mcp-*` / `acp-*`: protocolos de integración.
9. `server-*`, `service-*`, `daemon-*`, `websocket-*`: backend y transports.
10. `effect/*`, `kit/effect-*`, `layer-node-*`: intentos de modularización/refactor que revelan boundaries arquitectónicos.

## Limitación de esta primera pasada

Se ha verificado el número completo de ramas y se han inspeccionado en profundidad las ramas estructurales principales. Sin embargo, no sería correcto afirmar que las **1.595** ramas tienen ya una explicación individual confirmada únicamente por su nombre. Para las ramas efímeras restantes, el análisis definitivo debe cruzar commits exclusivos, merge-base, paths modificados y, cuando exista, PR asociado.

Este documento se diseñó para ser ampliado de forma incremental sin mezclar inferencias con hechos confirmados.
