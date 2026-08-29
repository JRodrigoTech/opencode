# Anthropic Messages, Gemini y Bedrock Converse

Estos tres protocolos muestran por qué OpenCode no puede reducir todos los providers a un dialecto OpenAI-compatible. Comparten un contrato normalizado aguas arriba, pero tienen reglas de wire format, continuidad y autenticación sustancialmente distintas.

---

# Anthropic Messages

## Request shape

Adapter: `packages/llm/src/protocols/anthropic-messages.ts`.

**[HECHO]** Soporta bloques nativos de:

- text;
- image base64;
- thinking;
- local `tool_use`;
- `server_tool_use`;
- server tool results;
- local `tool_result`.

**[HECHO]** Los tool results pueden ser string o secuencias text/image, preservando media de forma nativa.

## Thinking

**[HECHO]** Los bloques thinking incorporan una `signature`.

**[HECHO]** Durante raising, esa firma se almacena en `providerMetadata.anthropic.signature`; durante lowering posterior se recupera para continuar la conversación.

**[INFERENCIA]** El reasoning visible no es un transcript autosuficiente: la signature forma parte del estado de continuación del protocolo.

## Server-side tools

**[HECHO]** Anthropic diferencia `server_tool_use` de tools locales y devuelve resultados especializados como:

- `web_search_tool_result`;
- `code_execution_tool_result`;
- `web_fetch_tool_result`.

OpenCode los representa con `providerExecuted: true` y preserva el payload estructurado para round-trip.

## Tool/message ordering

La existencia de branches `anthropic-tool-order`, `anthropic-tool-finish`, `fix-anthropic-transform` y tests de malformed assistant tool order demuestra que el orden de bloques no es cosmético.

**[HECHO]** El protocolo contiene reparaciones/validaciones específicas para secuencias de mensajes y tools.

**[INFERENCIA]** El modelo normalizado de OpenCode debe ser más permisivo que el wire protocol; el adapter es responsable de producir una secuencia aceptable por Anthropic.

## System updates

**[HECHO]** `dev` sólo usa mid-conversation system messages nativos para el modelo exacto `claude-opus-4-8`.

Para otros modelos, el update se envuelve en un user-visible fallback bajo condiciones que no rompan la secuencia de tools.

**[INFERENCIA]** Esta lógica es una policy de compatibilidad model-specific dentro del protocol lowering, no una propiedad universal de Messages API.

## Cache control

**[HECHO]** Anthropic limita a cuatro breakpoints explícitos por request.

- `ephemeral` 5m por defecto;
- TTL largo bucketizado a 1h;
- breakpoints extra se eliminan y contabilizan.

**[HECHO]** El presupuesto se comparte entre tools/system/messages.

## Usage

Anthropic reporta:

- `input_tokens` no cacheados;
- `cache_creation_input_tokens`;
- `cache_read_input_tokens`;
- `output_tokens`.

**[HECHO]** OpenCode suma las tres categorías de input para construir `inputTokens` inclusivo.

**[PENDIENTE]** Anthropic no expone en este contrato un reasoning-token breakdown equivalente a OpenAI/Gemini; `reasoningTokens` queda sin derivación fiable.

## Auth

Provider facade: `packages/llm/src/providers/anthropic.ts`.

**[HECHO]** Usa header `x-api-key`, con configuración explícita o `ANTHROPIC_API_KEY`.

**[HISTÓRICO]** La línea Vertex Messages añade OAuth/ADC porque el mismo protocolo Messages puede ejecutarse sobre infraestructura Google Vertex con autenticación completamente diferente.

---

# Gemini

## Request shape

Adapter: `packages/llm/src/protocols/gemini.ts`.

**[HECHO]** Gemini usa roles `user`/`model`, y parts independientes para:

- text;
- inline data;
- function call;
- function response.

## Reasoning / thought signatures

**[HECHO]** Un text part puede contener:

- `thought: true`;
- `thoughtSignature`.

Los function calls también pueden llevar thought signature.

**[HECHO]** OpenCode la persiste en `providerMetadata.google.thoughtSignature` y la vuelve a insertar al continuar.

**[INFERENCIA]** Gemini confirma el mismo patrón observado en OpenAI/Anthropic: el contrato portable necesita un escape hatch para estado criptográfico/opaco del proveedor.

## Tool schema projection

**[HECHO]** Gemini no acepta JSON Schema arbitrario. La implementación realiza dos fases:

1. **sanitize**: arregla enums numéricos, `required` inválido, arrays sin `items`, propiedades incompatibles;
2. **project**: convierte a un subconjunto del dialecto Gemini.

Se descartan features no soportadas como `additionalProperties` o `$ref` cuando no forman parte del allowlist.

**[INFERENCIA]** El schema de tool portable es deliberadamente más expresivo que cada dialecto provider-specific. La compatibilidad se resuelve mediante proyección, no degradando el schema común al mínimo denominador.

## Tool results multimodales

**[HECHO]** Un tool result puede generar un `functionResponse` textual y, además, inline media parts.

## Sampling

`gemini-sampling` es una branch específica de esta superficie.

**[HECHO]** `dev` puede lower:

- max output tokens;
- temperature;
- topP;
- topK;
- stop sequences;
- thinking budget/include thoughts.

## Finish mapping

**[HECHO]** Gemini normaliza varias causas de safety como `content-filter`, incluyendo SAFETY, BLOCKLIST, PROHIBITED_CONTENT, SPII, RECITATION e IMAGE_SAFETY.

`MALFORMED_FUNCTION_CALL` se trata como error.

## Usage

Gemini reporta:

- `promptTokenCount`;
- `cachedContentTokenCount`;
- `candidatesTokenCount`;
- `thoughtsTokenCount`;
- `totalTokenCount`.

**[HECHO]** `candidatesTokenCount` representa output visible; OpenCode le suma `thoughtsTokenCount` para obtener `outputTokens` inclusivo.

## Error frames

La branch `gemini-error-frame` evidencia que algunos fallos se manifiestan dentro del stream/frame protocol y no sólo como HTTP no-2xx.

**[INFERENCIA]** Igual que Responses y Bedrock, Gemini necesita distinguir “transport opened” de “generation succeeded”.

---

# Bedrock Converse

## Protocolo

Adapter: `packages/llm/src/protocols/bedrock-converse.ts`.

**[HECHO]** Bedrock no usa SSE estándar. La respuesta se decodifica mediante AWS EventStream (`bedrock-event-stream.ts`).

**[HECHO]** El body usa:

- `modelId`;
- `messages`;
- `system`;
- `inferenceConfig`;
- `toolConfig`;
- `additionalModelRequestFields`.

## Content model

**[HECHO]** Soporta:

- text;
- image;
- document;
- tool use;
- tool result text/json/image;
- reasoning content;
- cache points.

## Reasoning

**[HECHO]** Bedrock transporta `reasoningContent.reasoningText.signature`.

OpenCode la guarda como provider metadata/encrypted reasoning y la reenvía al reconstruir history.

## Tool calling

**[HECHO]** Tool choice se traduce a:

- `{auto:{}}`;
- `{any:{}}`;
- `{tool:{name}}`.

Tool results tienen status `success`/`error` y pueden transportar JSON directamente.

## Cache points

**[HECHO]** Bedrock usa bloques posicionales `cachePoint` en lugar del `cache_control` inline de Anthropic.

**[HECHO]** La implementación reserva un máximo de cuatro para modelos Claude y los consume en orden tools → system → messages para favorecer los prefijos de mayor reutilización.

## Streaming errors

**[HECHO]** El stream conoce frames explícitos de fallo:

- `internalServerException`;
- `modelStreamErrorException`;
- `validationException`;
- `throttlingException`;
- `serviceUnavailableException`.

**[INFERENCIA]** La taxonomía común de errores existe precisamente para colapsar estos frames AWS y los errores HTTP de otros proveedores en un contrato comparable.

## Authentication

Provider facade: `packages/llm/src/providers/amazon-bedrock.ts`.

Dos modos:

1. bearer API key;
2. AWS SigV4.

### SigV4 en `dev`

**[HECHO]** `packages/llm/src/protocols/utils/bedrock-auth.ts` firma el request exacto con `aws4fetch` y service `bedrock`.

Credentials requeridas:

- region;
- accessKeyId;
- secretAccessKey;
- optional sessionToken.

**[HECHO]** El signer no refresca STS credentials. El comentario del código exige que el consumer reconstruya el modelo antes de expirar.

### Credential chain posterior

**[HECHO]** `bedrock-credential-chain` implementa en `core/model-resolver.ts` la cadena AWS default usando `@aws-sdk/credential-providers`.

Orden observado:

- configuración explícita existente;
- `AWS_BEARER_TOKEN_BEDROCK`;
- profile configurado o `AWS_PROFILE`;
- Node provider chain;
- `AWS_REGION` o default de región.

La función cachea el provider chain por profile, no las credenciales finales.

**[HECHO]** El comentario explica la intención: `ModelResolver` corre antes de cada provider turn, por lo que puede volver a resolver identidad temporal y luego construir una route con credenciales estáticas frescas.

**[INFERENCIA]** Esta es una decisión de boundary muy reveladora:

- **core/resolver** posee discovery/refresh de credenciales;
- **route auth** posee firma determinista del request exacto.

No se mezcla AWS credential lifecycle dentro del protocol parser.

---

# Comparación transversal

| Aspecto | Anthropic | Gemini | Bedrock |
|---|---|---|---|
| Wire protocol | Messages/SSE | Gemini streaming/SSE | Converse/AWS EventStream |
| Reasoning state | signature | thoughtSignature | reasoning signature |
| Tool dialect | tool_use/tool_result | functionCall/functionResponse | toolUse/toolResult |
| Server tools | Sí | no central en baseline | provider/model dependent |
| Inline caching | cache_control | no | cachePoint |
| Cache cap explícito | 4 | n/a | 4 en camino Claude |
| Auth baseline | x-api-key | x-goog-api-key | SigV4 o bearer |
| Usage reasoning | no breakdown explícito | thoughtsTokenCount | provider/model dependent |
| Tool schema | JSON schema cercano | proyección Gemini | JSON en inputSchema |

## Conclusión arquitectónica

**[INFERENCIA]** El “provider abstraction” real de OpenCode no intenta esconder diferencias semánticas importantes. Normaliza sólo lo que puede garantizar —messages, lifecycle, usage categories, tools, reasoning lifecycle y errors— y conserva en protocol adapters aquello que requiere reglas específicas. Eso evita dos fallos típicos de una abstracción LLM excesivamente genérica:

1. perder estado necesario para continuation;
2. afirmar capacidades que el wire protocol no puede representar de forma equivalente.
