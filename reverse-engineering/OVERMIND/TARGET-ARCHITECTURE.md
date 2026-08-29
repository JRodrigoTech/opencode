# Recommended Target Architecture

**Status:** RECOMMENDED — synthesis of VERIFIED-OVERMIND + VERIFIED-OPENCODE

## Target

```text
                         +----------------------+
                         | CLI / WebUI / ACP    |
                         | Runtime API adapters |
                         +----------+-----------+
                                    |
                                    v
                            +---------------+
                            | SessionService|
                            +--+---------+--+
                               |         |
                      +--------+         +--------+
                      v                           v
                RunState/Execution           Persistence
                      |
                      +---------------------> EventPort
                      |
                      v
                   Agent
          +-----------+------------+
          |            |           |
          v            v           v
 ContextCompiler  Capability   ModelService
  canonical        Resolver         |
  cognition           |             v
                      |       GenerationExecutor
                      v             |
                 ToolExecutor       v
                 /    |    \     ModelBackend
                /     |     \
        Permission  Journal  Plugin capability
                            /        |       \
                          MCP      Skills   Workspace

SubagentService -> creates child SessionService execution -> same Core Agent contract
```

## Ownership

### Agent

Sigue poseyendo el cognitive loop: model turn, Tool protocol atomicity y decisión de completar/continuar/fallar.

### ContextCompiler

Sigue siendo el único owner de assembly/attention/budget/compaction. No consulta una DB para decidir arbitrariamente qué contexto usar: recibe ContextUnits y ContextContributors según contratos explícitos.

### SessionService

Nuevo outer boundary. Posee identity, parent/child link, lifecycle y recuperación de session state. No conoce provider payloads.

### Execution / RunState

Posee active run, cancellation, state transitions y exclusión/parallelism permitido. Hace observable `idle/running/cancelling/failed/completed` sin entrar en semántica cognitiva.

### EventPort

Implementa el diseño FACT/SIGNAL ya definido por Overmind. La lección de OpenCode es producir eventos desde commit boundaries, no reconstruirlos desde logs.

### Persistence

Required sink para FACTs que deban ser durables. Guarda records versionados e idempotentes. Nunca transforma hechos para decidir Agent behavior.

### CapabilityResolver

Separa *registered capability* de *visible capability in this execution*. Inputs explícitos: agent/profile, target/model capabilities, permissions, client/runtime surface y session policy.

### ToolExecutor

Convierte Tool calls en lifecycle observable y autorizado. El ToolRegistry permanece frozen y simple.

### PermissionService

Policy runtime independiente del Plugin construction grant. Construction grants controlan qué puede recibir un plugin; PermissionService controla acciones concretas del agente/usuario.

### MutationJournal

Generaliza reversibilidad de Workspace a un contrato de side effects, sin asumir Git como Core dependency.

## Invariant crítico

```text
Persisted session facts
      |
      +----> audit / UI / resume / events
      |
      +----> reconstruction of canonical ContextUnits
                    |
                    v
              ContextCompiler
                    |
                    v
              provider messages[]
```

No debe existir el atajo `DB messages[] -> provider` como autoridad cognitiva.
