# OpenCode `dev`: baseline de ingeniería inversa

> Baseline de código analizado: `dev@dc4449df0d52199704ea4989a5a993ebbc605612`  
> Fecha del baseline: 2026-08-29  
> Rama de documentación: `reverse-engineering`  
> Alcance: arquitectura vigente del agente OpenCode y sus superficies de ejecución.

## Objetivo

Este directorio reconstruye cómo funciona realmente OpenCode en la branch `dev`, evitando un inventario superficial de archivos. El foco es el sistema en ejecución: cómo entra una petición, cómo se selecciona una instancia, cómo se materializa una sesión, cómo se construye una llamada al modelo, cómo se ejecutan tools, cómo se persiste el estado y cómo TUI/Desktop consumen el backend.

## Convención de evidencia

- **[CONFIRMADO]**: comportamiento observado directamente en código del baseline indicado.
- **[INFERENCIA]**: interpretación arquitectónica apoyada por varios hechos, pero no expresada literalmente como contrato en el código.
- **[PENDIENTE]**: punto que requiere ejecución dinámica, tests o análisis de otra branch para cerrarse.

Todas las referencias a paths se entienden contra `dev@dc4449df0d52199704ea4989a5a993ebbc605612`, salvo indicación contraria.

## Resultado ejecutivo

**[CONFIRMADO]** OpenCode es un monorepo Bun/TypeScript con un composition root todavía muy importante en `packages/opencode`, pero con responsabilidades ya extraídas a paquetes especializados: `@opencode-ai/core`, `@opencode-ai/llm`, `@opencode-ai/server`, `@opencode-ai/tui`, `@opencode-ai/schema`, `@opencode-ai/protocol`, SDK y aplicaciones cliente.

**[CONFIRMADO]** El agente no funciona como `prompt -> LLM -> respuesta`. El runtime central es una máquina de ejecución alrededor de una **sesión persistente**, con reconstrucción de historial, compaction, selección dinámica de agente/modelo, resolución de tools y permisos, streaming del modelo, ejecución de tool calls, actualización incremental de mensajes/parts, retries y repetición del loop hasta alcanzar un estado terminal.

**[CONFIRMADO]** El server/API es un boundary central. TUI y Desktop se comportan como clientes del backend incluso cuando todo corre localmente. La TUI normal ejecuta el backend en un `Worker` y lo consume mediante un transporte `fetch`/event source sobre RPC; Desktop supervisa un sidecar local y conecta el renderer a ese servidor.

**[CONFIRMADO]** La persistencia vigente es híbrida. Existe SQLite/Drizzle con WAL y tablas normalizadas, un pipeline de eventos/proyecciones y, simultáneamente, compatibilidad V1/V2 y storage legacy. `SessionProjector` proyecta eventos de sesión a tablas SQLite, mientras el nuevo modelo introduce eventos durables con secuencia por aggregate.

**[INFERENCIA]** `dev` representa una migración arquitectónica incremental desde un núcleo monolítico en `packages/opencode` hacia boundaries más explícitos y reutilizables. La coexistencia de Session V1/V2, server local + `@opencode-ai/server`, provider orchestration + `@opencode-ai/llm`, y adapters de compatibilidad indica una estrategia de extracción progresiva, no una reescritura total.

## Índice

1. [Baseline y metodología](./00-baseline-y-metodologia.md)
2. [Arquitectura y boundaries](./01-arquitectura-y-boundaries.md)
3. [Entrypoints y runtime del agente](./02-entrypoints-y-runtime.md)
4. [Sesiones, eventos y persistencia](./03-sesiones-eventos-y-persistencia.md)
5. [Configuración, tools y providers](./04-configuracion-tools-y-providers.md)
6. [Server, backend y protocolos](./05-server-backend-y-protocolos.md)
7. [TUI, Desktop y clientes](./06-tui-desktop-y-clientes.md)
8. [Conclusiones, riesgos y preguntas abiertas](./07-conclusiones-riesgos-y-preguntas.md)

## Mapa de lectura recomendado

Para entender el agente desde dentro hacia fuera: `02` → `03` → `04` → `05` → `06`. Para una visión de arquitectura primero: `01` → `05` → `02`.

## Paths de alta señal

| Área | Path / símbolo |
|---|---|
| CLI composition root | `packages/opencode/src/index.ts` |
| Loop del agente | `packages/opencode/src/session/prompt.ts` (`SessionPrompt`) |
| Consumo del stream | `packages/opencode/src/session/processor.ts` (`SessionProcessor`) |
| Resolución de tools | `packages/opencode/src/session/tools.ts` |
| Tool registry | `packages/opencode/src/tool/registry.ts` |
| Providers/modelos | `packages/opencode/src/provider/provider.ts` (`Provider`) |
| Adaptación LLM | `packages/opencode/src/session/llm.ts` (`LLM`) |
| Configuración | `packages/opencode/src/config/config.ts` (`Config`) |
| Sesiones V1 | `packages/opencode/src/session/session.ts` |
| Bridge de eventos | `packages/opencode/src/event-v2-bridge.ts` |
| Event runtime | `packages/core/src/event.ts` (`EventV2`) |
| Proyección SQL | `packages/core/src/session/projector.ts` (`SessionProjector`) |
| Schema SQL | `packages/core/src/session/sql.ts` |
| Database | `packages/core/src/database/database.ts` |
| Server composition | `packages/opencode/src/server/routes/instance/httpapi/server.ts` |
| Server listener | `packages/opencode/src/server/server.ts` |
| TUI command/worker | `packages/opencode/src/cli/cmd/tui.ts`, `packages/opencode/src/cli/tui/worker.ts` |
| TUI package | `packages/tui/src/app.tsx` |
| Desktop main | `packages/desktop/src/main/index.ts` |

## Snapshot reproducible

Además del commit de baseline, varios blobs críticos observados son:

- `packages/opencode/src/index.ts`: `13540a73a36fcb27f591011d9147c60073055b45`
- `packages/opencode/src/session/prompt.ts`: `0f85d44f209ba792065aeb951f0bd2e12b59fae8`
- `packages/opencode/src/session/processor.ts`: `20aa8a8404d8e5f50b0aafee5034ed6f1fa44382`
- `packages/opencode/src/session/tools.ts`: `0f401c7562fa07076afd539990ca12fa207ceee0`
- `packages/opencode/src/provider/provider.ts`: `b5980f15873b22647b03aa75fe450e2344aed5b9`
- `packages/opencode/src/config/config.ts`: `9e10b67fe703609a3ccf243ebe1801bf338961d3`
- `packages/opencode/src/server/server.ts`: `440b992c155774c9611dd01b3de2f400a522e71b`
- `packages/opencode/src/server/routes/instance/httpapi/server.ts`: `fb9d2db65621540a514786f2061714f16b6c766c`
- `packages/opencode/src/tool/registry.ts`: `9167cb3ea6bc5c8dd075f0f8271adbdec6074b12`

Esto permite detectar cambios posteriores incluso si `dev` avanza.