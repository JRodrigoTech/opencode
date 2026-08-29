# OpenCode -> Overmind Extraction Matrix

## Leyenda

- **PRESERVE** — Overmind ya posee el boundary adecuado.
- **DELEGATE** — mantener la capacidad en OpenCode y consumirla desde Overmind.
- **ADAPT IF NEEDED** — extraer solo la primitive genérica cuando exista requisito real.
- **REFERENCE / DEFER** — estudiar, no implementar ahora.
- **REJECT** — copiarlo degradaría Overmind.

## Matriz

| Mecanismo OpenCode | Decisión para Overmind | Motivo |
|---|---|---|
| `build` / `plan` / `explore` coding agents | **DELEGATE** | Son perfiles especializados de software engineering; Overmind puede invocarlos. |
| `TaskTool` + subagents de OpenCode | **DELEGATE** | Permitir que el coding agent organice su propia exploración/implementación; no replicar la jerarquía. |
| shell/bash | **DELEGATE** para coding | Mantener ejecución de comandos compleja dentro del boundary especializado. |
| read/edit/write/apply_patch | **DELEGATE** para coding | Overmind conserva Workspace Tools simples; no necesita duplicar el editor de OpenCode. |
| grep/glob/code search/LSP | **DELEGATE** | Infraestructura específica de codebase. |
| Code Mode | **DELEGATE / REFERENCE** | Si aporta valor, que lo use OpenCode internamente; no es una primitive cognitiva universal. |
| Git snapshots/revert de coding | **DELEGATE** | OpenCode puede poseer reversibilidad de su sesión; no obliga a un MutationJournal Core. |
| project instructions / AGENTS / CLAUDE para coding | **DELEGATE** | OpenCode puede interpretar instrucciones del repo; Overmind no necesita duplicar ese discovery. |
| `opencode run --format json` | **ADAPT NOW** | Boundary mínimo para una Tool de coding sin nuevos subsistemas Core. |
| `opencode acp` | **ADAPT WHEN RICH LIFECYCLE IS NEEDED** | Excelente transporte para external-agent sessions, permission forwarding, resume/cancel. |
| OpenCode ACP server internals | **DELEGATE** | El adapter consume el protocolo; no copia su implementación. |
| ACP server para exponer Overmind | **REFERENCE / DEFER** | Es otra dirección del protocolo; esperar Runtime API real. |
| external agent session ID | **ADAPT MINIMALLY** | Tratarlo inicialmente como opaque delegation reference; no obliga a SessionService propio. |
| parent/child Session domain | **REFERENCE / ADAPT IF NEEDED** | Útil si Overmind implementa subagents generales; no necesario para delegar coding. |
| BackgroundJob | **REFERENCE / ADAPT IF NEEDED** | OpenCode ya puede hacer background/subagents internos; Overmind solo necesita su propio background si hay un caso general. |
| Permission engine `allow/ask/deny` | **REFERENCE** | Mantener outer grant + inner OpenCode policy; genericizar solo con una necesidad transversal real. |
| normalized LLM events | **REFERENCE / ADAPT IF NEEDED** | Útil para UI/observability futura, no requisito para integrar coding. |
| Session persistence | **REFERENCE / ADAPT IF NEEDED** | Overmind necesitará persistence por sus propios objetivos, no para imitar OpenCode. |
| Event stream / reducer | **REFERENCE / ADAPT IF NEEDED** | Adoptar solo cuando WebUI/services/background requieran un event contract general. |
| MCP | **INDEPENDENT OVERMIND CAPABILITY** | Si Overmind necesita MCP, implementarlo como Plugin según sus contracts; no copiarlo porque OpenCode lo tenga. |
| Skills | **PRESERVE OVERMIND DESIGN** | Encajan mejor como ContextContributors/TOOLS bounded que como prompt mutation. |
| plugin before/after hooks | **REJECT GENERAL HOOK SURFACE** | Preservar public ports; typed middleware solo con caso probado. |
| provider registry + provider transforms | **REJECT FOR OVERMIND CORE** | OpenCode mantiene sus providers; Overmind ya posee ModelBackend seam. |
| AI SDK/native dual runtime | **REJECT STRUCTURE** | No aporta valor al stack Python de Overmind. |
| Effect Layers | **REJECT** | Tecnología específica del runtime TypeScript. |
| model-name tool heuristics | **REJECT** | Preferir capabilities explícitas. |
| messages/parts como source primario de sesión | **REJECT COMO CONTEXT AUTHORITY** | Overmind conserva canonical ContextUnits. |
| HTTP server/generated clients | **REFERENCE / DEFER** | Solo cuando una interface remota de Overmind lo necesite. |

## Qué sí se extrae inmediatamente

La extracción inmediata útil es pequeña:

1. **delegation as a high-level Tool**;
2. **opaque external session reference** para reanudar cuando sea útil;
3. **bounded result contract**;
4. **safe subprocess/ACP boundary**;
5. **separación entre outer delegation policy e inner coding permissions**.

El resto de la sofisticación de OpenCode puede permanecer detrás del adapter.

## Qué ya tiene Overmind y no debe rehacerse

- ContextCompiler y budgeting.
- canonical ContextUnits/ProtocolUnits.
- ModelService/GenerationExecutor.
- ToolRegistry frozen.
- Plugin staging/validation/freeze.
- Workspace confinement/recovery.
- retry/recovery separation.

## Regla final

```text
Universal mechanism needed by Overmind -> adapt minimally
Domain-specialized coding behavior      -> delegate to OpenCode
Existing Overmind invariant             -> preserve
OpenCode complexity without requirement -> defer/reject
```
