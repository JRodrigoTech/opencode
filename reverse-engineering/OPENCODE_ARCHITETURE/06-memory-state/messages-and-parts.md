# Messages and Parts Persistence

**Status:** VERIFIED-CODE

## Storage unit

OpenCode persiste messages y parts de forma separada. `Session.getMessage`/streaming histórico reconstituye `WithParts` para el runtime.

## Why parts matter

Parts representan el execution transcript:

- texto y reasoning;
- tool calls/resultados;
- step boundaries;
- patches;
- compaction markers;
- files/resources;
- subtask/agent references.

Esto permite a clientes consumir progreso incremental y al loop reconstruir estado sin inferirlo desde texto.

## Update pattern

`SessionProcessor` usa `session.updatePart`, `updatePartDelta` y `updateMessage` durante streaming. Las parts no se escriben solo al final de la respuesta.

## Message lineage

Assistant messages referencian `parentID` del user message que contestan. El loop busca latest user/assistant comparando IDs/timestamps de la secuencia persistida.

## Model projection vs persisted representation

No todas las parts se envían literalmente al provider. `MessageV2.toModelMessagesEffect` proyecta la historia al formato de modelo y puede serializar tools/attachments de forma específica.

## Compaction awareness

`filterCompactedEffect(sessionID)` produce la historia utilizable por el loop. Esto hace que la persistencia completa pueda contener más información que el contexto activo.

## Sources

- `packages/opencode/src/session/message-v2.ts`
- `packages/opencode/src/session/session.ts`
- `packages/opencode/src/session/processor.ts`
