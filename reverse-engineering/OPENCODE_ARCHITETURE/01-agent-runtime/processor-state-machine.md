# SessionProcessor State Machine

**Status:** VERIFIED-CODE

`SessionProcessor` es la state machine que transforma el stream normalizado del LLM en estado persistente.

## Processor context

Cada handle mantiene:

- assistant message actual;
- `toolcalls` por call ID;
- `shouldBreak`;
- snapshot actual;
- `blocked`;
- `needsCompaction`;
- text part activo;
- `reasoningMap` por reasoning ID.

El snapshot inicial se captura **antes** de arrancar el stream porque el AI SDK puede ejecutar tools internamente antes de emitir un `step-start`.

## Event transitions

### Reasoning

`reasoning-start` crea un `ReasoningPart`; `reasoning-delta` concatena texto y emite deltas persistentes; `reasoning-end` marca tiempo de fin y actualiza metadata. Deltas huérfanos sin start se descartan.

### Text

`text-start` abre un `TextPart`; `text-delta` actualiza contenido incremental; `text-end` ejecuta el hook `experimental.text.complete`, fija timestamps y persiste la versión final.

### Tool input/call

`tool-input-start/delta/end` aseguran la existencia de un `ToolPart`. `tool-call` cambia el estado a running con input y metadata.

Los estados relevantes del tool part son:

```text
pending -> running -> completed
                 └-> error
```

`tool-result` normaliza output/metadata/attachments. Las imágenes adjuntas se normalizan; las que no pueden ajustarse al límite se omiten y el output se anota.

### Doom-loop detection

Para cada `tool-call` se inspeccionan los últimos `DOOM_LOOP_THRESHOLD = 3` parts del assistant. Si son la misma tool con input JSON idéntico y no pending, se solicita permiso `doom_loop`. El mecanismo no aborta arbitrariamente; transfiere la decisión al policy engine.

### Step start/finish

`step-start` asegura snapshot y persiste un `step-start` part.

`step-finish`:

1. toma snapshot posterior;
2. finaliza reasoning pendiente;
3. calcula usage y cost;
4. fija `assistant.finish`;
5. persiste `step-finish`;
6. calcula patch respecto al snapshot previo;
7. si hay ficheros modificados, persiste `PatchPart`;
8. lanza summary;
9. marca `needsCompaction` si hay overflow.

## Cleanup invariant

`cleanup` se ejecuta siempre al finalizar el procesamiento. Cierra text/reasoning incompletos, espera brevemente tool calls, marca las restantes como error `Tool execution aborted` con `metadata.interrupted=true`, completa timestamp del assistant y persiste.

Ese marker `interrupted` se usa posteriormente en el loop para distinguir tool calls abandonadas de trabajo pendiente legítimo.

## Process result

Tras stream/retry/cleanup:

- `needsCompaction` → `compact`
- `blocked` o assistant error → `stop`
- en otro caso → `continue`

## Sources

- `packages/opencode/src/session/processor.ts` — `20aa8a8404d8e5f50b0aafee5034ed6f1fa44382`
