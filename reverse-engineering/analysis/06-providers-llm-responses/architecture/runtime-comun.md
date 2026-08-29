# Runtime común de LLM y providers

## Resumen

**[HECHO]** En `dev`, el paquete `packages/llm` implementa un runtime propio y portable. El flujo no ejecuta directamente un SDK de proveedor; construye un `LLMRequest`, resuelve una `Route`, baja el request al wire format del protocolo, aplica autenticación/transporte y vuelve a elevar el stream a `LLMEvent`.

## Boundaries reales

### Provider facade

Ejemplos:

- `packages/llm/src/providers/openai.ts`
- `packages/llm/src/providers/anthropic.ts`
- `packages/llm/src/providers/google.ts`
- `packages/llm/src/providers/azure.ts`
- `packages/llm/src/providers/amazon-bedrock.ts`
- `packages/llm/src/providers/xai.ts`

**[HECHO]** La facade concentra defaults del proveedor, auth sugar, base URL, query params y selección de routes. No implementa el parser del stream.

**[INFERENCIA]** La facade es deliberadamente delgada para que un provider pueda cambiar endpoint/auth sin bifurcar el protocolo.

### Route

`packages/llm/src/route/client.ts` actúa como boundary de composición.

**[HECHO]** `compile()`:

1. resuelve opciones/defaults;
2. aplica cache policy;
3. toma la route del modelo;
4. ejecuta `route.body.from(resolved)`;
5. valida el body con schema;
6. ejecuta `route.prepareTransport`;
7. devuelve request, route, body y payload preparado.

No hace I/O del provider.

**[HECHO]** `generate()` se implementa reduciendo el stream normalizado hasta `LLMResponse`, no mediante una segunda vía de ejecución.

### Protocol

**[HECHO]** Un protocol define el mapping bidireccional:

- request portable → body nativo;
- chunks/eventos nativos → `LLMEvent`.

Esto se ve claramente en `openai-responses.ts`, `anthropic-messages.ts`, `gemini.ts` y `bedrock-converse.ts`.

### Transport

**[HECHO]** El transporte está separado del protocolo. OpenAI Responses puede usar HTTP o WebSocket con el mismo contrato lógico. Bedrock requiere un framing AWS EventStream específico.

## Model resolution

### Estado en `dev`

`packages/core/src/session/runner/model.ts` resuelve un modelo del catálogo hacia un `@opencode-ai/llm` Model.

**[HECHO]** Los mappings nativos confirmados son:

- `@ai-sdk/openai` → OpenAI Responses;
- `@ai-sdk/anthropic` → Anthropic Messages;
- `@ai-sdk/openai-compatible` con URL explícita → OpenAI-compatible Chat.

**[HECHO]** El resolver mezcla:

- selección de variant;
- overlay de headers/body;
- credential metadata;
- auth;
- endpoint override;
- limits de contexto/output.

**[PENDIENTE]** El paquete LLM soporta más providers que los que el resolver `dev` puede seleccionar desde metadata `aisdk`.

### Dirección posterior

En branches como `bedrock-credential-chain` aparece `packages/core/src/model-resolver.ts`.

**[HISTÓRICO]** Ese resolver introduce un resultado explícito:

- `model`: runtime provider model;
- `ref`: identidad durable del catálogo;
- `capabilities`;
- `cost`.

**[INFERENCIA]** Esto formaliza que la identidad de catálogo y el model id enviado al proveedor son conceptos distintos.

## Authentication

`packages/llm/src/route/auth.ts` define una pequeña álgebra de autenticación.

**[HECHO]** Soporta:

- secret literal;
- secret desde `Config`;
- bearer;
- header custom;
- composición `andThen`;
- fallback `orElse`;
- auth custom effectful;
- eliminación de headers previos.

**[HECHO]** Los secretos se representan como `Redacted` durante resolución.

### Ejemplos

- OpenAI: bearer, `OPENAI_API_KEY`.
- Anthropic: `x-api-key`, fallback `ANTHROPIC_API_KEY`.
- Google Gemini: `x-goog-api-key`.
- Azure: elimina `authorization` y usa `api-key`.
- Bedrock: bearer API key o SigV4.

## Request executor y retry

`packages/llm/src/route/executor.ts` implementa el boundary HTTP compartido.

**[HECHO]** Constantes relevantes:

- `MAX_RETRIES = 2`;
- `BASE_DELAY_MS = 500`;
- `MAX_DELAY_MS = 10_000`;
- body de error retenido hasta 16 KiB.

**[HECHO]** Retry por status incluye 429, 503, 504 y 529; también puede reutilizar `Retry-After` / `Retry-After-Ms`.

**[HECHO]** El error normalizado distingue, entre otros:

- authentication;
- insufficient permissions;
- quota exceeded;
- rate limit;
- invalid request;
- context overflow;
- content policy;
- provider internal;
- transport;
- unknown provider error.

**[HECHO]** El executor extrae request ids comunes de OpenAI, AWS, Google y Cloudflare.

## Redacción de secretos

**[HECHO]** Se redactan headers, query params y campos de bodies cuyo nombre parezca secreto/token/api-key/credential/signature.

**[HECHO]** También se reemplazan literalmente valores secretos enviados por el request si el provider los refleja en el body de error.

**[INFERENCIA]** La observabilidad de provider failures está diseñada para poder persistirse/reportarse sin filtrar credenciales por defecto.

## Event contract

`packages/llm/src/schema/events.ts` define el stream portable.

### Texto

- `text-start`
- `text-delta`
- `text-end`

### Reasoning

- `reasoning-start`
- `reasoning-delta`
- `reasoning-end`

### Tools

- `tool-input-start`
- `tool-input-delta`
- `tool-input-end`
- `tool-call`
- `tool-result`
- `tool-error`

### Lifecycle

- `step-start`
- `step-finish`
- `finish`
- `provider-error`

**[HECHO]** `LLMResponse` se reconstruye a partir de esos eventos; por tanto el evento normalizado es el contrato primario y la respuesta completa es una proyección.

## Tool execution boundary

**[HECHO]** `providerExecuted: true` diferencia tools ejecutadas por el proveedor de tools locales del agente.

Ejemplos provider-side:

- OpenAI Responses: web search, file search, code interpreter, computer use, image generation, MCP, local shell.
- Anthropic: `server_tool_use` y bloques server tool result.

**[INFERENCIA]** Esta bandera evita que el agent runtime vuelva a ejecutar una tool que ya fue ejecutada por el proveedor y permite persistir ambos casos bajo un mismo modelo de mensajes.

## Reasoning continuity

**[HECHO]** El contrato de reasoning conserva metadata provider-specific en vez de asumir que el texto visible basta para replay.

- OpenAI: item id + encrypted reasoning content.
- Anthropic: signature.
- Gemini: thought signature.
- Bedrock: signature dentro de reasoning content.

**[INFERENCIA]** `providerMetadata` funciona como un side channel de continuidad y auditoría: la UI/runtime puede operar sobre un contrato común, pero el siguiente request puede recuperar el estado opaco que exige el wire protocol.

## Cache policy

`packages/llm/src/cache-policy.ts` aplica caching antes del lowering.

**[HECHO]** Default `auto`:

- breakpoint tras última tool definition;
- breakpoint tras último system part;
- breakpoint en el último mensaje de usuario.

**[HECHO]** Sólo se inyectan hints para `anthropic-messages` y `bedrock-converse`.

**[HECHO]** Hints manuales existentes no se sobrescriben.

### Provider lowering

**Anthropic**

- máximo 4 cache breakpoints explícitos;
- TTL se bucketiza a 5m/1h;
- extras se descartan y se contabilizan.

**Bedrock**

- cache points posicionales dentro de tools/system/messages;
- usa el mismo presupuesto de cuatro breakpoints para modelos Claude vía Bedrock.

**OpenAI/Gemini**

- no usan estos markers inline;
- OpenAI expone `prompt_cache_key` y usage de cached tokens;
- Gemini reporta `cachedContentTokenCount`.

## Usage normalization

`Usage` establece una convención común.

**[HECHO]** Totales inclusivos:

- `inputTokens` incluye cache reads/writes;
- `outputTokens` incluye reasoning;
- `totalTokens` usa el total del provider o suma input+output.

**[HECHO]** Breakdown no solapado:

- `nonCachedInputTokens`;
- `cacheReadInputTokens`;
- `cacheWriteInputTokens`;
- `reasoningTokens`.

### Diferencias de wire protocol

**OpenAI**

El provider ya reporta input/output inclusivos y subsets de cached/reasoning.

**Anthropic**

`input_tokens` representa la parte no cacheada; el mapper suma cache creation/read para reconstruir el total inclusivo. No hay breakdown fiable de thinking separado de output.

**Gemini**

`candidatesTokenCount` es output visible; se suma `thoughtsTokenCount` para reconstruir `outputTokens` inclusivo.

**Bedrock**

Expone input/output/cache read/cache write en usage de Converse.

## Cost accounting

**[HECHO]** El runtime LLM no contiene una tabla de precios ni calcula billing contractual.

**[HISTÓRICO]** En el `ModelResolver` posterior, el coste del catálogo viaja como `Resolved.cost` separado del usage.

**[INFERENCIA]** La arquitectura pretende calcular coste fuera de los protocol adapters: adapter produce usage; catálogo aporta pricing; una capa superior combina ambos.

## Error ownership

**[HECHO]** Hay dos fuentes principales:

1. errores HTTP/transport generados por `RequestExecutor`;
2. errores emitidos dentro del stream por el protocolo, convertidos a `provider-error` o fallo normalizado.

**[INFERENCIA]** Esta separación es necesaria para APIs como Responses/Anthropic/Bedrock, donde un HTTP 200 puede abrir correctamente el stream y el error aparecer posteriormente como evento de protocolo.
