# OpenAI Responses, Open Responses y continuidad

## Estado en `dev`

El adapter principal es `packages/llm/src/protocols/openai-responses.ts`.

**[HECHO]** La route HTTP usa:

- base URL `https://api.openai.com/v1`;
- path `/responses`;
- streaming;
- el mismo modelo portable de messages/tools/reasoning que el resto del runtime.

**[HECHO]** OpenAI provider expone tres caminos sobre una misma facade:

- Responses por HTTP;
- Responses por WebSocket;
- Chat Completions.

La selección no cambia el contrato consumido por las capas superiores.

## Request lowering

### Mensajes

**[HECHO]** El request portable se transforma a items de Responses:

- system → item de role `system`;
- user text/image → `input_text` / `input_image`;
- assistant text → `output_text`;
- tool call → `function_call`;
- tool result → `function_call_output`;
- reasoning → reasoning item o `item_reference`.

**[HECHO]** Los tool results que contienen imágenes conservan media provider-native; no se convierten en base64 embebido dentro de un string JSON.

**[INFERENCIA]** Esto evita inflación artificial de contexto y preserva multimodalidad hasta el wire protocol.

### Tools

**[HECHO]** El schema de tools se proyecta explícitamente al dialecto que acepta OpenAI. `strict` está actualmente fijado a `false` en el lowering de `dev`.

**[HECHO]** `tool_choice` portable se traduce a `auto`, `none`, `required` o una función concreta.

## Reasoning

OpenAI Responses requiere más estado que el texto del razonamiento.

**[HECHO]** `providerMetadata.openai` conserva:

- `itemId`;
- `reasoningEncryptedContent`.

### `store !== false`

**[HECHO]** Si Responses mantiene server-side state, el replay puede usar:

```text
item_reference(id)
```

para reasoning items y hosted tool items ya conocidos.

### `store === false`

**[HECHO]** En modo stateless el adapter reconstruye el reasoning item con summaries y `encrypted_content`.

**[HECHO]** Si no existe encrypted state completo, el reasoning replay se filtra antes de enviar el siguiente request, porque OpenAI no acepta ese item parcial en `store:false`.

**[INFERENCIA]** La continuidad de reasoning en Responses es un problema de identidad + estado opaco, no un simple concatenado de contenido visible.

## Identidad y replay

La familia de branches `responses-call-identity` → `responses-item-replay` → `responses-reasoning` evidencia la estabilización progresiva de esta parte del protocolo.

**[HECHO]** `responses-item-replay`, comparada con una base cercana de la generación `remove-ai-sdk`, concentra cambios en:

- `open-responses.ts`;
- `openai-responses.ts`;
- lifecycle;
- tests de reasoning continuation e item replay.

**[INFERENCIA]** `responses-call-identity` antecede conceptualmente al replay: sin una identidad estable entre `item.id`, `call_id` y los IDs normalizados del runtime no es posible reconstruir tool loops o reasoning de forma segura.

## Hosted tools

**[HECHO]** `dev` reconoce items provider-executed de Responses, incluyendo:

- web search;
- web search preview;
- file search;
- code interpreter;
- computer use;
- image generation;
- MCP;
- local shell.

El adapter emite para ellos el mismo par lógico que para tools normales:

```text
tool-call(providerExecuted=true)
tool-result(providerExecuted=true)
```

**[HECHO]** El payload nativo completo se conserva como structured result/provider metadata cuando es útil.

**[INFERENCIA]** Este diseño desacopla el agent loop del lugar físico donde se ejecutó la herramienta: local runtime y provider runtime comparten event model, pero no execution ownership.

## Stream parser

**[HECHO]** El parser mantiene estado para:

- lifecycle de texto/reasoning;
- tool input incremental;
- presencia de function calls;
- reasoning items y summary parts;
- flag `store`.

### Reasoning summaries

**[HECHO]** La implementación conoce el orden de eventos de summary parts e identifica cada bloque con `item_id:summary_index`.

**[HECHO]** El código declara explícitamente que el comportamiento ante eventos fuera de orden es best-effort, no una garantía.

### Terminación

**[HECHO]** Son terminales:

- `response.completed`;
- `response.incomplete`;
- `response.failed`.

Los dos primeros generan finish normalizado; `response.failed` genera provider failure.

## Finish reason

**[HECHO]** La normalización de Responses traduce:

- sin incomplete reason + function call → `tool-calls`;
- sin function call → `stop`;
- `max_output_tokens` → `length`;
- `content_filter` → `content-filter`.

## Usage

**[HECHO]** OpenAI reporta:

- `input_tokens` inclusivo;
- subset `cached_tokens`;
- `output_tokens` inclusivo;
- subset `reasoning_tokens`.

El mapper deriva `nonCachedInputTokens` y conserva usage bruto bajo `providerMetadata.openai`.

## Opciones específicas

Campos explícitamente soportados en `dev` incluyen:

- `store`;
- `service_tier`;
- `prompt_cache_key`;
- `include`;
- reasoning effort;
- reasoning summary;
- text verbosity;
- max output tokens;
- temperature;
- top-p.

**[HISTÓRICO]** La matriz de `remove-ai-sdk` todavía marcaba la cobertura tipada como parcial y señalaba structured output y continuation incremental como áreas incompletas.

## WebSocket

**[HECHO]** `dev` expone una route WebSocket específica de Responses además de HTTP.

**[HISTÓRICO]** En ramas posteriores aparecen cassettes/escenarios que prueban:

- continuar un tool call sobre un único socket;
- reconstruir contexto tras reconexión;
- recuperarse de rechazo de una continuación explícita.

**[INFERENCIA]** La arquitectura evita hacer del WebSocket una sesión durable del agente. El canal es un mecanismo de transporte/continuación del provider; la sesión durable sigue perteneciendo a core.

## De OpenAI Responses a Open Responses

En ramas posteriores, sobre todo `remove-ai-sdk`, `openresponses-audit` y las líneas de Responses, la implementación se divide en:

- `open-responses.ts`: protocolo base provider-neutral;
- `openai-responses.ts`: especialización OpenAI;
- `openai-compatible-responses.ts`: deployment compatible;
- `open-responses-continuation.ts`: política/estado de continuación;
- `open-responses-channel.ts`: canal persistente/reutilizable a nivel de transport.

**[HECHO]** La matriz histórica separa explícitamente “OpenAI Responses” de “Open Responses-compatible”. El segundo no hereda por defecto tools, metadata ni defaults OpenAI.

**[INFERENCIA]** Esta separación corrige una abstracción demasiado amplia: que dos vendors expongan `/responses` no implica que tengan las mismas extensiones, hosted tools ni políticas de state.

## Relación con xAI y Vertex

**[HECHO]** En `dev`, xAI puede reutilizar `OpenAIResponses.route` con provider/baseURL propios.

**[HISTÓRICO]** La generación `remove-ai-sdk` añade Vertex Responses como facade distinta sobre Open Responses para Grok deployments y fuerza `store:false` como default de Vertex.

**[INFERENCIA]** La selección correcta se modela como:

```text
provider/deployment -> route -> protocol
```

no como:

```text
model vendor -> protocol fijo
```

Esto explica branches como `vertex-xai-default` y `deepseek-responses`.

## Retry

**[HECHO]** Hay dos dominios de fallo:

1. HTTP/transport antes o durante apertura, manejado por `RequestExecutor`;
2. errores dentro del stream Responses.

**[INFERENCIA]** `responses-stream-retry` y los timeout fixes posteriores exploran precisamente el límite entre estos dominios: repetir una petición HTTP idempotente no es equivalente a reanudar un stream con identidad/estado de provider ya creado.

## Ideas integradas frente a ideas pendientes

### Integradas en `dev`

- Responses como protocolo nativo;
- HTTP + WebSocket;
- reasoning metadata;
- item references;
- stateless reasoning replay;
- hosted tool events;
- usage normalizado;
- failure normalization;
- provider-specific options.

### Evolución posterior/no consolidada completamente en `dev`

- protocolo Open Responses totalmente provider-neutral;
- policy de continuation extraída a módulo propio;
- channel manager más rico;
- selección API-aware desde todo el catálogo;
- structured output provider-native completo.
