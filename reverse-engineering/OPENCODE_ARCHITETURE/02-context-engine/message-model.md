# Message Model

**Status:** VERIFIED-CODE + DERIVED

La conversación persistida no es una lista plana de strings. Cada message tiene info tipada y una lista de parts heterogéneos.

## Roles

La lógica central distingue al menos:

- `user`
- `assistant`

Los assistant messages mantienen además provider/model, agent/mode, token usage, cost, finish, errors, path, timing y opcionalmente structured output/summary.

## Important part families

A lo largo del runtime se observan parts para:

- `text`
- `reasoning`
- `tool`
- `file`
- `agent`
- `subtask`
- `step-start`
- `step-finish`
- `patch`
- `compaction`

Esto permite que la historia capture no solo lenguaje natural, sino ejecución y mutaciones del workspace.

## Tool part state

El estado de tool representa el lifecycle observable:

- pending: llamada detectada/preparándose;
- running: input aceptado y ejecución activa;
- completed: output, metadata, attachments y end time;
- error: input, error, metadata parcial y end time.

`providerExecuted` en metadata distingue tool calls ejecutadas por el provider/workflow de las que requieren el ciclo normal de OpenCode.

`interrupted=true` diferencia tool calls abandonadas por abort de trabajo que debe continuar.

## Synthetic parts

OpenCode introduce parts sintéticos para representar operaciones que deben quedar en contexto sin provenir literalmente del usuario, por ejemplo:

- resultado de resolver un file part mediante Read;
- instrucción de delegación a agent/task;
- resumen de una command subtask.

## Model projection

`MessageV2.toModelMessagesEffect(messages, model)` transforma la representación persistente a `ModelMessage[]`. Esa proyección es el boundary antes de provider-specific message transforms.

## Compacted projection

El loop utiliza `MessageV2.filterCompactedEffect(sessionID)` en vez de consumir ciegamente todas las filas históricas. La memoria efectiva depende de compaction/pruning.

## Sources

- `packages/opencode/src/session/message-v2.ts` — `9b3f2c46f40578128001957004c67633a18da23a`
- `packages/opencode/src/session/processor.ts`
- `packages/opencode/src/session/prompt.ts`
