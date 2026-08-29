# Inventario y evolución de branches

## Método

Se hizo discovery por prefijos y términos (`provider`, `native-provider`, `llm`, `responses`, `openai`, `anthropic`, `azure`, `bedrock`, `vertex`, `gemini`, `deepseek`, `xai`, `ai-native`, reasoning/stream/cache/usage), y después se contrastaron las líneas arquitectónicas principales contra `dev` y, cuando el merge-base hacía el diff demasiado ruidoso, contra una branch hermana de la misma generación para aislar el cambio funcional.

**[HECHO]** Varias branches tardías divergen cientos o miles de commits de `dev`. Un `compare dev...branch` sigue siendo útil para ubicar genealogía, pero no permite atribuir todos los cambios del diff a la feature indicada por el nombre. Por eso, para microfeatures se usa también comparación entre siblings cercanos.

## Familia 1 — Provider model y configuración

Branches descubiertas:

- `provider-benchmark`
- `provider-cleanup`
- `provider-config-merge`
- `provider-config-shape`
- `provider-credentials`
- `provider-entrypoints`
- `provider-error-data`
- `provider-event-tolerance`
- `provider-optional-fields`
- `provider-packages`
- `provider-readiness`
- `facade/provider`
- `feat/request-aware-provider-fetch`
- `fix/provider-session-failures`
- `native-provider-clean`
- `native-provider-config`
- `native-provider-core`
- `native-provider-packages`
- `native-provider-schema`
- `native-provider-stack`
- `refactor/core-provider-turn`
- `refactor/provider-inline-local-helpers`
- `refactor-provider-model-status-schema`
- `session-provider-errors`

### Lectura evolutiva

**[HECHO]** `native-provider-*` forma una línea coherente: schema/config/core/packages/stack/clean. El diff de `native-provider-stack` contra `dev` muestra cambios coordinados en provider schema, model/catalog, plugins, session runner y provider packages de `packages/llm`.

**[INFERENCIA]** Esta línea representa la transición desde “el catálogo contiene nombres de paquetes AI SDK y core los carga” hacia “el catálogo resuelve una identidad/provider package nativo y el runtime recibe un modelo ya construido”.

**[HECHO]** `provider-packages` / `provider-entrypoints` y el stack posterior introducen entrypoints provider-specific, en vez de depender de un registry único.

**[INFERENCIA]** `provider-config-*` y `provider-credentials` son refactors de ownership: configuración declarativa en catálogo/integration, pero semántica de auth concreta en facade/route.

## Familia 2 — Construcción del runtime LLM propio

Branches descubiertas:

- `llm-centralization`
- `llm-error-redesign`
- `llm-error-types`
- `llm-native-event-adapter`
- `llm-native-inject-client`
- `llm-native-prepare-tests`
- `llm-native-request-adapter`
- `llm-native-runtime-openai`
- `llm-remove-provider-error`
- `llm-service-event-seam`
- `llm-terminal-contract`
- `fix-native-llm-options`
- `kit/llm-facade-cleanup`
- `kit/session-llm-instance-state`
- `test-llm-dx`
- `worktree-draft+llm-usage-additive`
- `ai-native-routing`
- `openai-compatible-native-ai`
- `remove-ai-sdk`

### Generaciones

#### A. Centralización temprana

**[HISTÓRICO]** `llm-centralization` trabaja todavía alrededor de una capa de sesión más monolítica (`packages/opencode/src/session/llm.ts` en esa generación). Es un antecedente conceptual, no la arquitectura final.

#### B. Seams nativos

**[INFERENCIA]** `llm-native-request-adapter`, `llm-native-event-adapter`, `llm-native-inject-client`, `llm-service-event-seam` y `llm-native-runtime-openai` descomponen el problema en request adapter, event adapter, client injection y runtime. El shape resultante es reconocible en `dev`: schema portable + route/protocol + stream normalizado.

#### C. Contrato robusto

**[INFERENCIA]** `llm-error-types`, `llm-error-redesign`, `llm-terminal-contract`, `llm-remove-provider-error` muestran una fase posterior de estabilización del contrato de errores y terminación. `dev` ya centraliza errores de transporte/HTTP y mantiene `provider-error` como evento del stream.

#### D. Sustitución de AI SDK

**[HECHO]** `remove-ai-sdk` contiene un stack nativo mucho más amplio bajo `packages/ai` y una matriz explícita de paridad. Incluye OpenAI, Anthropic, Gemini, Vertex, Bedrock, Azure, OpenRouter, xAI, Cloudflare y compatibles.

**[INFERENCIA]** El nombre expresa el objetivo final, pero el propio `STATUS.md` demuestra que la sustitución no estaba completa: core seguía cayendo a AI SDK para múltiples catalog packages.

## Familia 3 — OpenAI Responses / Open Responses

Branches descubiertas:

- `responses-api-adjustments`
- `responses-call-identity`
- `responses-done-message`
- `responses-item-replay`
- `responses-reasoning`
- `responses-stream-retry`
- `deepseek-responses`
- `grok-responses-docs`
- `openresponses-audit`
- `openai-websocket`
- `fix/openai-stateless-reasoning-continuation`
- `fix/openai-websocket-header-timeout`
- `fix/openai-codex-route-timeouts`
- `openai-text-phase`

### Secuencia funcional inferida

1. API shape y finalización (`responses-api-adjustments`, `responses-done-message`).
2. Identidad estable de calls/items (`responses-call-identity`).
3. Replay explícito de items (`responses-item-replay`).
4. Reasoning y estado opaco (`responses-reasoning`, stateless continuation fix).
5. Retry de stream y terminal semantics (`responses-stream-retry`).
6. WebSocket y continuidad/reconexión (`openai-websocket`, timeout fixes).
7. Generalización provider-neutral (`openresponses-audit`).

**[HECHO]** En generaciones posteriores a `dev` aparecen módulos separados `open-responses.ts`, `open-responses-continuation.ts` y `open-responses-channel.ts`, mientras `openai-responses.ts` queda como especialización OpenAI.

**[INFERENCIA]** La evolución va de “adapter OpenAI” a “protocolo Responses provider-neutral + especializaciones de deployment”. Esto permite reutilizar Responses en Vertex/xAI/compatibles sin heredar accidentalmente defaults o hosted tools exclusivos de OpenAI.

## Familia 4 — Anthropic

Branches:

- `anthropic-fixes`
- `anthropic-thinking-variants`
- `anthropic-tool-finish`
- `anthropic-tool-order`
- `fix-anthropic-transform`
- `nxl/fix-anthropic-provider-tools`
- `deepseek-anthropic`
- `fix/google-vertex-anthropic-auth-message`
- `fix/google-vertex-anthropic-thinking`

**[HECHO]** `dev` conserva lógica explícita para thinking signatures, server tools, ordering, cache-control y una compatibilidad especial de system updates.

**[INFERENCIA]** Las branches de ordering/finish/transform evidencian que Anthropic no puede tratarse como un OpenAI-like: la validez del message sequence y la firma del thinking forman parte del protocolo.

## Familia 5 — Azure

Branches:

- `azure-chat-reasoning`
- `azure-cli-auth`
- `azure-cognitive-improvements`
- `azure-reasoning-cleanup`
- `fix-azure-issue`

**[HECHO]** En `dev`, Azure no tiene parser propio: especializa OpenAI Responses y OpenAI Chat con base URL, query `api-version` y `api-key`.

**[INFERENCIA]** Las ramas `azure-*reasoning` son principalmente compatibilidad/selection entre Chat y Responses, mientras `azure-cli-auth` apunta a ampliar la obtención de credenciales fuera del simple API key.

## Familia 6 — Bedrock

Branches:

- `bedrock-credential-chain`
- `bedrock-credentials`
- `bedrock-metadata-terminal`
- `bedrock-region`
- `fix-bedrock-1`

**[HECHO]** `dev` implementa Converse, AWS EventStream y SigV4 con credenciales estáticas proporcionadas a la route.

**[HECHO]** `bedrock-credential-chain` mueve la AWS default credential chain al `ModelResolver`: cachea el provider chain por profile y resuelve credenciales antes de cada provider turn, refrescando de facto las credenciales al reconstruir el modelo.

**[INFERENCIA]** `bedrock-region` y `bedrock-credentials` convergen en un boundary: resolución AWS en core/provider-package; firma exacta de bytes en protocol auth.

## Familia 7 — Gemini y Vertex

Gemini:

- `gemini-error-frame`
- `gemini-sampling`
- `gemini-thinking-level`
- `zen-only-gemini`

Vertex:

- `vertex-fixes`
- `vertex-multiregion`
- `vertex-xai-default`
- `fix/google-vertex-anthropic-auth-message`
- `fix/google-vertex-anthropic-thinking`
- `fix/google-vertex-metadata-warning`

**[HECHO]** Gemini en `dev` conserva thought signatures y realiza una proyección propia del JSON Schema de tools.

**[HECHO]** La línea posterior de Vertex separa al menos cuatro APIs sobre Google Cloud: Gemini, OpenAI-compatible Chat, Open Responses y Anthropic Messages.

**[HECHO]** `vertex-multiregion` calcula endpoint a partir de project/location, soporta tuned `endpoints/...`, Express Mode API key y OAuth/ADC; los modelos tuned no aceptan Express Mode API key.

**[INFERENCIA]** Vertex demuestra de forma especialmente clara la separación provider/protocol: una sola plataforma de autenticación/routing aloja varios protocolos incompatibles.

## Familia 8 — DeepSeek y xAI

DeepSeek:

- `deepseek-anthropic`
- `deepseek-responses`
- `deepseek-weekends`
- `fix-deepseek-reasoner`

xAI:

- `xai-capacity-text`
- `xai-capacity-v2`
- `xai-device-default`
- `xai-oauth`
- `fix/xai-provider-upstream-pdf-support`
- `vertex-xai-default`

**[HECHO]** `dev` incluye un profile DeepSeek dentro de OpenAI-compatible Chat. xAI dispone de Responses y Chat compatibles.

**[INFERENCIA]** Las ramas DeepSeek prueban que “OpenAI-compatible” es un family resemblance, no una equivalencia total: reasoning y nuevas APIs requieren profiles/adapters específicos.

**[INFERENCIA]** `vertex-xai-default` revela además que la elección del protocolo depende no sólo del vendor del modelo, sino del deployment que lo sirve.

## Familia 9 — Cache, usage y coste

Branches de interés:

- `cache-friendly-compaction`
- `cache-key-defaults`
- `cache-tail-validation`
- `reported-usage-cost`
- `worktree-draft+llm-usage-additive`
- `fix/zen-openai-response-usage`

**[HECHO]** `dev` normaliza usage de forma aditiva/inclusiva y separa pricing del protocolo.

**[INFERENCIA]** La evolución converge en dos reglas: evitar restas ambiguas en accounting de tokens, y tratar cache layout como parte de request compilation/context strategy, no como concern exclusivo del provider facade.

## Qué llegó a `dev`

Confirmado en `dev`:

- runtime LLM independiente;
- protocol adapters nativos;
- route/auth/endpoint/transport desacoplados;
- OpenAI Responses + Chat;
- Anthropic Messages;
- Gemini;
- Bedrock Converse;
- Azure y xAI como facades sobre protocolos reutilizados;
- reasoning metadata portable con escape hatch provider-specific;
- tools provider-executed;
- cache policy común;
- usage normalizado;
- typed provider failures y retry centralizado.

## Qué no está plenamente integrado en `dev`

**[HECHO]** El resolver de catálogo de `dev` no selecciona nativamente toda la superficie soportada por `packages/llm`.

**[HISTÓRICO/PENDIENTE]** En `remove-ai-sdk`, la matriz de paridad todavía marcaba huecos en Azure token auth, Vertex catalog mapping, Bedrock default credential chain y structured output nativo.

**[INFERENCIA]** Las ideas descartadas no son principalmente los protocolos nativos; lo descartado o retrasado fue activar todos esos protocolos como default del core antes de alcanzar paridad operativa y de credenciales con AI SDK.
