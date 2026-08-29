# OVERMIND — OpenCode Cross-Architecture Study

> **Objetivo:** extraer de OpenCode mecanismos útiles para la evolución de Overmind sin degradar sus boundaries cognitivos actuales.  
> **OpenCode baseline:** `JRodrigoTech/opencode@production:df35e842f59bc115bb7c0479a8e11f017d443f2c`  
> **Overmind baseline:** `JRodrigoTech/Overmind@main:c8b68b55b4a057232c764cafc545d4633fbeb22f`  
> **Documentación:** `JRodrigoTech/opencode@reverse-engineering/reverse-engineering/OVERMIND/`

## Propósito

Esta carpeta no propone convertir Overmind en OpenCode. Los dos sistemas tienen objetivos y decisiones fundacionales diferentes.

OpenCode es un coding-agent runtime maduro, stateful y orientado a sessions/tools/providers/interfaces. Overmind está construyendo un **cognitive runtime modular** donde el LLM es un motor de razonamiento dentro de un sistema mayor y donde el contexto canónico pertenece al Core, no al transport `messages[]` del provider.

La comparación se usa con cuatro etiquetas:

- **VERIFIED-OVERMIND** — comportamiento actual confirmado en source/tests/docs normativas de Overmind.
- **VERIFIED-OPENCODE** — mecanismo confirmado en la baseline production de OpenCode.
- **RECOMMENDED** — adaptación propuesta para Overmind; no está implementada.
- **DO-NOT-COPY / DEFER** — idea que no conviene importar ahora o debe permanecer diferida.

## Hallazgo principal

Lo más valioso de OpenCode para Overmind no es su estructura de packages ni sus decisiones TypeScript/Effect. Son sus mecanismos operativos acumulados:

1. Session identity y parent/child execution.
2. Runtime events normalizados y un reducer de ejecución.
3. Tool lifecycle observable con IDs, metadata, cancellation y permission checks.
4. Permission engine separado de tool discovery.
5. Subagents como child sessions reanudables, con model/policy/budget propios.
6. Background jobs con lifecycle y entrega de resultados al parent.
7. Persistencia de execution history sin depender del LLM para reconstruir estado.
8. Snapshots/patches/revert como journal de mutaciones.
9. MCP/skills/protocol adapters fuera del agent loop.
10. Interfaces externas (HTTP/ACP/client) apoyadas sobre un runtime interno único.

Overmind ya tiene decisiones que conviene **preservar por encima de OpenCode**:

- Context canónico separado de provider messages.
- `ContextContributor -> ContextBlock -> ContextFrame -> ContextCompiler -> CompiledContext`.
- Tool protocol almacenado como unidades atómicas.
- `ModelService -> GenerationExecutor -> ModelBackend` con retry físico separado de recovery semántico.
- composition staging + validation + freeze.
- grants explícitos a plugins; no acceso al object graph completo.
- FACT vs SIGNAL como diseño futuro de EVENTS.

## Arquitectura objetivo recomendada

```text
Interaction / Runtime API
        |
        v
   SessionService -----------------------> Persistence
        |                                      ^
        v                                      |
 RunState / Execution -----> RuntimeEventBus --+
        |
        v
      Agent
   /    |     \
  /     |      \
ContextCompiler  CapabilityResolver  ModelService
(canonical)          |                  |
                     v                  v
                ToolExecutor      GenerationExecutor
                  |    |                 |
                  |    +-> Permission    v
                  |                ModelBackend
                  +-> MutationJournal
                  +-> Plugins / MCP
                  +-> ChildSession/Subagent
```

**Regla central:** Session/Persistence/EventBus no deben convertir `messages[]` en memoria cognitiva autoritativa. Persisten hechos y unidades; `ContextCompiler` sigue decidiendo qué entra al modelo.

## Mapa de documentos

- `AUDIT-BASELINE.md` — commits congelados, autoridad y límites.
- `CURRENT-STATE.md` — qué existe realmente hoy en Overmind.
- `SOURCE-MAP.md` — hotspots Overmind/OpenCode usados.
- `EXTRACTION-MATRIX.md` — copiar/adaptar/defer/rechazar por subsistema.
- `TARGET-ARCHITECTURE.md` — arquitectura futura propuesta.
- `PRIORITY-ROADMAP.md` — prioridad y dependencias.
- `DO-NOT-COPY.md` — anti-patterns que no deben importarse.
- `00-context/` — protección del ContextCompiler.
- `01-runtime/` — sessions, events, run-state y cancellation.
- `02-tools-permissions/` — capabilities, permissions y lifecycle.
- `03-subagents/` — child sessions y background execution.
- `04-state/` — persistence y reversibility.
- `05-extensibility/` — MCP, Skills y Plugin boundaries.
- `06-model/` — normalización de model runtime y multi-provider.
- `07-interfaces/` — Runtime API, ACP y server.
- `08-implementation/` — fases y acceptance gates.

## Orden recomendado

1. `CURRENT-STATE.md`
2. `EXTRACTION-MATRIX.md`
3. `TARGET-ARCHITECTURE.md`
4. `PRIORITY-ROADMAP.md`
5. `DO-NOT-COPY.md`
6. documentos de cada subsistema según la fase de implementación.
