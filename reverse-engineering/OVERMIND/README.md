# OVERMIND — Qué extraer de OpenCode sin convertir Overmind en OpenCode

> **OpenCode baseline:** `JRodrigoTech/opencode@production:df35e842f59bc115bb7c0479a8e11f017d443f2c`  
> **Overmind baseline:** `JRodrigoTech/Overmind@main:c8b68b55b4a057232c764cafc545d4633fbeb22f`  
> **Documentación:** `JRodrigoTech/opencode@reverse-engineering/reverse-engineering/OVERMIND/`

## Decisión principal

Esta carpeta **no diseña un clon de OpenCode dentro de Overmind**.

Overmind es un cognitive runtime general y modular. OpenCode es un agente especializado en software engineering. Cuando Overmind necesite una capacidad de coding compleja, la opción preferida es **delegarla a un agente OpenCode** mediante una Tool/adaptador de subagente, no importar a Overmind el stack de shell, edit, patch, LSP, Git, prompts de coding, routing de tools, provider quirks y subagents de OpenCode.

La relación objetivo es:

```text
Overmind cognition
      |
      | complete high-level ToolCall
      v
agent.delegate / opencode.delegate
      |
      v
OpenCode adapter
  CLI JSON initially
  ACP when richer lifecycle is needed
      |
      v
OpenCode coding agent
  + sessions
  + build/plan/explore
  + shell/read/edit/patch/LSP
  + permissions
  + internal subagents/background work
  + provider/model adaptations
      |
      v
bounded DelegationResult
      |
      v
Overmind ProtocolUnit / ContextCompiler
```

**OpenCode es un capability provider especializado, no una dependencia del Core cognitivo de Overmind.**

## Regla de adopción

Cada mecanismo de OpenCode se clasifica en una de cuatro categorías:

- **PRESERVE-OVERMIND** — Overmind ya posee un boundary mejor alineado con su objetivo; se mantiene.
- **DELEGATE-TO-OPENCODE** — la capacidad es específica de coding y debe permanecer dentro del agente OpenCode.
- **ADAPT-WHEN-REQUIRED** — el concepto es genérico y valioso, pero solo se promueve a Overmind cuando una necesidad propia lo justifique.
- **DO-NOT-COPY / REFERENCE-ONLY** — no aporta valor suficiente o introduciría complejidad prematura.

La existencia de una feature en OpenCode **no es evidencia suficiente para implementarla en Overmind**.

## Qué debe permanecer en Overmind

La baseline de Overmind ya establece propiedades que esta comparación no debe degradar:

- `ContextContributor -> ContextBlock -> ContextFrame -> ContextCompiler -> CompiledContext`.
- `messages[]` como transport output, nunca memoria/contexto canónico.
- `ModelService -> GenerationExecutor -> ModelBackend`.
- retry técnico separado de recovery/continuation semántico.
- Tool protocol fail-closed y `ProtocolUnit` atómica.
- Plugins por public ports, staging/validation/freeze y grants explícitos.
- Core pequeño: mecanismos universales, no capacidades de dominio.

## Qué coding NO necesita reconstruir Overmind

Mientras OpenCode sea el backend especializado, Overmind no necesita duplicar:

- agentes `build`, `plan`, `explore` o `general` de coding;
- shell/bash;
- glob/grep/code search;
- read/edit/write/apply_patch;
- LSP;
- Code Mode;
- Git snapshot/revert de sesiones de coding;
- prompts e instrucciones específicas de repositorios;
- compatibilidad provider/model propia de coding;
- subagents internos de OpenCode.

Overmind puede entregar una misión de coding a OpenCode y permitir que **OpenCode use sus propios subagentes** dentro de esa misión.

## Dos niveles de integración

### MVP: Tool de delegación

Una `OpenCodePlugin` puede aportar una única Tool de alto nivel y ejecutar `opencode run --format json`. La propia CLI soporta agent, model, directory y reanudación por session ID. Este camino permite validar el valor sin crear nuevos subsistemas Core.

### Integración rica: ACP client

`opencode acp` inicia un Agent Client Protocol server sobre stdio. La implementación production expone session lifecycle, prompt, resume, fork, cancel, mode/model y permission requests. Si Overmind necesita streaming estructurado, permissions interactivas, cancelación o sesiones persistentes del agente externo, un adapter ACP es el boundary natural.

**ACP client hacia OpenCode** y **ACP server para exponer Overmind** son problemas distintos. El primero puede ser útil temprano; el segundo sigue siendo una interfaz futura de Overmind.

## Mapa de documentos

- `ADOPTION-STRATEGY.md` — reglas para decidir qué integrar, delegar o rechazar.
- `EXTRACTION-MATRIX.md` — clasificación feature por feature.
- `TARGET-ARCHITECTURE.md` — boundary recomendado Overmind -> agente externo.
- `PRIORITY-ROADMAP.md` — orden basado en valor real, no en paridad con OpenCode.
- `DO-NOT-COPY.md` — límites explícitos.
- `REVISION-NOTES.md` — corrección de la primera versión de este estudio.
- `03-subagents/OPENCODE-AS-CODING-AGENT.md` — diseño técnico detallado de delegación.
- resto de carpetas — lecciones genéricas de Context, Tools, state, extensibilidad, model e interfaces, siempre bajo la regla de adopción anterior.

## Lectura recomendada

1. `ADOPTION-STRATEGY.md`
2. `EXTRACTION-MATRIX.md`
3. `03-subagents/OPENCODE-AS-CODING-AGENT.md`
4. `TARGET-ARCHITECTURE.md`
5. `PRIORITY-ROADMAP.md`
6. `DO-NOT-COPY.md`

Para hechos de OpenCode usar `../OPENCODE_ARCHITETURE/` como baseline auditada; para autoridad de Overmind prevalecen source/tests y `AGENTIX` del repositorio Overmind.
