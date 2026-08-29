# Preserve the Overmind Context Architecture

**Decision:** KEEP / STRENGTHEN

## Overmind tiene una ventaja importante

`ContextCompiler` no es un serializer de chat history. Es un policy engine cognitivo que controla authority, required/optional material, exact token budgets, compaction y final rendering.

OpenCode construye system/messages de forma efectiva para un coding agent, pero Overmind pretende incorporar Memory, RAG, Blackboard y background cognition. Para ese objetivo, `ContextBlock`/`ContextFrame` es el boundary más escalable.

## Cómo integrar Session persistence

Persistence no debe guardar solo provider messages. Debe poder reconstruir canonical units:

```text
Session storage
  - UserUnit record
  - AssistantUnit record
  - ProtocolUnit record
  - Context summary state/version
  - execution metadata/events
        |
        v
ContextCompiler.compile(...)
```

Los ContextContributors siguen produciendo blocks por request. Memory/RAG/Blackboard no escriben directamente provider messages.

## OpenCode lessons compatibles

### Context overflow is runtime state

OpenCode trata overflow como una transición que puede producir compaction y retry del loop. Overmind ya tiene mejor accounting/preflight; puede añadir un FACT de pressure/compaction sin cambiar ownership.

### Instructions are discovered capabilities

La idea de OpenCode de project/user instructions puede mapearse a un Overmind `ContextContributor` con authority/required semantics explícitas. No insertar strings fuera del compiler.

### Skills

Una Skill puede ser:

```text
Skill catalog
 -> selected skill descriptor
 -> ContextContributor / ContextBlock
 -> ContextCompiler budget
```

Esto integra skills mejor que anexarlas directamente al system prompt.

## Context refactor actual

Los propios docs de Overmind identifican que `compiler.py` y `compaction.py` necesitan split funcional, manteniendo `ContextCompiler` como único owner de estado mutable. Esa refactorización debería ocurrir antes o paralela a persistence, para evitar que un nuevo Session layer empiece a depender de privados del compiler.

## Invariants a congelar con tests

- persisted resume produce las mismas canonical ContextUnits;
- current required material nunca se trunca silenciosamente;
- Tool ProtocolUnit se conserva atómica;
- persisted summary no duplica source units al provider;
- event/persistence metadata nunca aumenta authority de un ContextBlock;
- provider Thinking no entra automáticamente en Context.
