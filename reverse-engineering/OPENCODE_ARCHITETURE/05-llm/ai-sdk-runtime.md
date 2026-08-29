# AI SDK Runtime

**Status:** VERIFIED-CODE

## Default path

El camino AI SDK utiliza `streamText` y es el runtime de compatibilidad general.

## Model wrapping

Se usa middleware `wrapLanguageModel` para interceptar `transformParams`. En cada llamada:

`ProviderTransform.message(input.model, input.params, opts)`

puede modificar prompt/options según provider/model.

## Key `streamText` inputs

Entre los campos observados:

- `onError` con logging;
- `temperature`, `topP`, `topK`;
- `providerOptions`;
- `activeTools`;
- `tools`;
- `toolChoice`;
- `maxOutputTokens`;
- abort signal;
- headers;
- OpenTelemetry experimental telemetry;
- system prompt;
- messages.

## Tool call repair

`experimental_repairToolCall` intenta corregir nombres cuando el modelo invoca una tool con diferencia de case. Si no puede reparar, construye una llamada a `invalid` con el error original y lista de nombres válidos.

## Stop condition

El `streamText` interno usa `stopWhen: stepCountIs(1)`. Esto es intencionado: OpenCode quiere controlar el ciclo multistep en su propio `SessionPrompt.runLoop`, no delegarlo enteramente al AI SDK.

## Event conversion

`createAISDKEventStream` convierte el full stream del SDK al vocabulario `LLMEvent` utilizado por `SessionProcessor`.

## Architectural implication

OpenCode usa AI SDK principalmente como **provider execution adapter**, pero conserva ownership del agent loop, persistence, tools y retries.

## Source

- `packages/opencode/src/session/llm.ts`
