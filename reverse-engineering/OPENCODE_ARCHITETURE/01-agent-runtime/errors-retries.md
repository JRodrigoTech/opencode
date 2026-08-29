# Errors and Retries

**Status:** VERIFIED-CODE

## Error ownership

Los errores atraviesan varias capas, cada una con distinta responsabilidad:

- **Provider/transport**: errores HTTP, headers/SSE timeouts, provider errors.
- **LLM adapter**: convierte streams/SDK errors a eventos o fallos Effect.
- **SessionRetry**: decide si y cómo reintentar.
- **SessionProcessor**: transforma errores terminales a `SessionV1` errors y publica `Session.Event.Error`.
- **Tool layer**: tool errors se representan en `ToolPart.state=error`.

## Retry placement

`SessionProcessor.process` aplica `Effect.retry(SessionRetry.policy(...))` alrededor del drenaje del stream. La policy recibe:

- provider ID;
- función `parse` para convertir el error;
- callback `set` que actualiza `SessionStatus` a `retry` con attempt/message/action/next.

Por tanto, el estado observable de la session distingue busy normal de espera por retry.

## Context overflow

`halt` trata `ContextOverflowError` especialmente.

- Si `compaction.auto === false` y no se está generando un summary, el error queda en el assistant, finish=`error`, se publica evento y la session pasa a idle.
- En otro caso marca `needsCompaction=true`; el processor devuelve `compact` y el loop crea una compaction.

Overflow es así una transición del control loop, no únicamente una excepción.

## Content filter

Tras `process`, `SessionPrompt` detecta `finish === "content-filter"`, crea `ContentFilterError`, actualiza mensaje, publica error y rompe el loop. Evita que una respuesta bloqueada sin texto visible parezca finalización limpia.

## Tool errors

`tool-error` o un `tool-result` de tipo error llaman `failToolCall`. Se preserva metadata ya emitida mientras estaba running. Si el error procede de rechazo de permiso/pregunta, puede marcar el processor como `blocked` dependiendo de `continue_loop_on_deny`.

## Invalid tool calls

El AI SDK path configura `experimental_repairToolCall`. Si el nombre solo difiere en case y existe una tool lowercase, repara el nombre. En caso contrario redirige a la tool `invalid` con información del error de parsing/validación.

## Cancellation vs failure

Abort se distingue de error no-interruptivo. La cleanup marca llamadas pendientes como interrupted; el loop las ignora al decidir si debe re-prefill al assistant. Esto evita bucles posteriores causados por tool calls abandonadas.

## Sources

- `packages/opencode/src/session/processor.ts`
- `packages/opencode/src/session/retry.ts` — `284c0f0ade4143df6aae127e60c511f5466c46a6`
- `packages/opencode/src/session/llm.ts`
- `packages/opencode/src/provider/error.ts`
