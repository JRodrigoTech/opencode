# Target Architecture — composición con agentes especializados

**Status:** RECOMMENDED, minimal-first

## Target inmediato

El target no añade una copia del runtime de OpenCode alrededor del Core de Overmind. Añade una capability delegada dentro del Plugin/Tool model existente.

```text
                    OVERMIND

Request -> Agent -> ContextCompiler -> ModelService
            |
            | complete ToolCall
            v
         ToolPort
            |
            +---------------- WorkspacePlugin (simple deterministic file tools)
            |
            +---------------- OpenCodePlugin
                                  |
                                  v
                         OpenCodeDelegateTool
                                  |
                         transport adapter
                       /                  \
          MVP: `opencode run`       richer: ACP client
                       \                  /
                                  v
                          OPENCODE PROCESS
                    coding session / coding agent
                      /      |       |       \
                  tools   permissions Git   subagents
                                  |
                                  v
                         DelegationResult
                                  |
                                  v
                       Overmind ProtocolUnit
```

## Ownership

### Overmind Agent

Posee la decisión cognitiva de delegar y el Tool protocol. No conoce OpenCode sessions, providers o tools internas.

### ContextCompiler

Sigue poseyendo todo el contexto de Overmind. Una delegación es una observation/protocol result más, no una puerta para insertar la transcript externa completa.

### OpenCodePlugin

Posee configuración y disponibilidad de la capability `opencode`. Puede registrar una Tool de alto nivel. El Core no importa OpenCode.

### OpenCode adapter

Posee process invocation/protocol translation. Convierte un `DelegationInput` a una misión OpenCode y normaliza el resultado. Los IDs de sesión OpenCode permanecen opaque fuera de este boundary.

### OpenCode

Posee completamente su runtime de software engineering: agents, prompts, tools, permissions, subagents, provider stack, snapshots y coding state.

## Contrato mínimo

```text
DelegationInput
- task: str
- capability/profile: optional
- workspace scope: configured, not arbitrary by default
- external_session_id?: str
- selected context/artifact refs?: bounded

DelegationResult
- ok/status
- summary
- external_session_id
- changed/artifact refs when available
- verification/test summary when available
- bounded metadata/error
```

La primera versión puede modelarse directamente como una Tool. No necesita un nuevo Core port.

## Evolución opcional a AgentDelegationPort

Solo cuando exista un segundo backend de agentes o semántica común real:

```text
AgentDelegationPort
        |
        +-- OpenCodeAgentAdapter
        +-- LocalOvermindSubagentAdapter
        +-- OtherExternalAgentAdapter
```

Entonces se pueden estabilizar operaciones como `delegate`, `resume`, `cancel` y eventualmente background. Hasta ese momento, mantenerlas plugin-local reduce superficie y evita diseñar un framework hipotético.

## Subagente Overmind != agente OpenCode

Un futuro subagente nativo de Overmind puede usar el mismo Core cognitivo con context seed/budgets propios. Eso no significa que todas las especialidades deban implementarse como subagentes nativos.

Para coding:

```text
Overmind
  -> external OpenCode coding agent
       -> OpenCode internal explore/general/... subagents
```

Para una futura capacidad cognitiva propia de Overmind:

```text
Overmind
  -> local Overmind subagent
```

Ambos pueden converger detrás de un contrato de delegación **solo si esa convergencia aporta valor real**.

## Permission boundary

```text
Overmind outer policy
- may delegate to opencode?
- workspace/profile allowed?
- data allowed to leave parent context?
        |
        v
OpenCode inner policy
- read/edit/shell/web permissions
- subagent permissions
- external directory rules
```

No copiar el permission engine de OpenCode para esta integración.

## Persistence

Para el MVP basta devolver `external_session_id` en la Tool observation. Si después Overmind implementa persistence por Memory/WebUI/resume general, puede persistir un `DelegationRecord` con esa referencia opaque. No almacenar la transcript OpenCode como canonical Overmind memory.

## Independence invariant

`OpenCodePlugin` eliminado => desaparece la capacidad especializada de coding, pero el resto de Overmind continúa funcionando sin cambios en Core.
