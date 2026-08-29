# Context Compaction

**Status:** VERIFIED-CODE

## Goal

Compaction reduce el contexto enviado al LLM sin eliminar necesariamente la historia persistente completa. Produce un summary semántico y conserva una cola reciente.

## Important constants

En la baseline se observan thresholds como:

- `PRUNE_MINIMUM = 20_000`
- `PRUNE_PROTECT = 40_000`
- serialización de tool output para compaction truncada alrededor de 2.000 caracteres;
- presupuesto de cola reciente limitado por defaults aproximadamente entre 2.000 y 15.000 tokens.

Los valores exactos deben releerse si cambia el blob `75d6374b...`.

## History selection

El algoritmo separa:

- `head`: historia antigua candidata a resumen;
- `tail`: mensajes recientes preservados íntegramente dentro del budget.

Puede reutilizar summaries de compactions anteriores para evitar volver a resumir desde cero.

## Hidden compaction agent

La generación del summary usa el agent oculto `compaction`, sin tools. El prompt de compaction solicita conservar contexto necesario para continuar la tarea.

## Pruning

Además de resumir, pruning puede sustituir/eliminar outputs de tools antiguos cuando acumulan suficiente peso. Hay tools protegidas, entre ellas `skill`, cuyos outputs no se podan de la misma forma.

## Trigger paths

Compaction puede activarse por:

- overflow calculado al final de un step;
- error de context overflow cuando auto-compaction está permitida;
- solicitudes explícitas/maintenance del runtime.

## Plugin interception

Antes de finalizar el prompt de compaction se ejecuta `experimental.session.compacting`, permitiendo modificar context/prompt del resumen.

## What OpenCode remembers

La memoria efectiva del modelo en un turno es:

```text
compaction summaries
+ recent uncompressed tail
+ current instructions/system context
+ newly resolved tool/file/resource material
```

No equivale a “todos los mensajes de la base de datos”.

## Sources

- `packages/opencode/src/session/compaction.ts` — `75d6374bfa54e5f492c2f0be83fa3029794009eb`
- `packages/opencode/src/agent/agent.ts`
