# AGENT 06 — Providers, LLM runtime y protocolos

## Objetivo

Este árbol reconstruye la arquitectura de providers/LLM de OpenCode y su evolución a partir de `dev` y de las familias de branches relacionadas con providers, protocolos, reasoning, streaming, Responses API y retirada de AI SDK.

Baseline: `dev@dc4449df0d52199704ea4989a5a993ebbc605612`.

## Corrección de auditoría: dos consumidores del stack LLM

`dev` contiene un stack nativo sustancial en `packages/llm`, pero **el runtime de producto de `packages/opencode` no ha sustituido universalmente AI SDK por ese stack**.

Hay que distinguir:

1. **SessionPrompt/product path** — `packages/opencode/src/session/llm.ts`.
   - obtiene el `LanguageModel` mediante `Provider.Service`;
   - por defecto ejecuta `ai.streamText(...)`;
   - normaliza el `fullStream` hacia `LLMEvent`;
   - puede intentar `LLMNativeRuntime.stream(...)` sólo cuando `experimentalNativeLlm` está habilitado;
   - si el modelo no está soportado por el runtime nativo, cae explícitamente a AI SDK.
2. **Core V2 path** — `packages/core/src/session/runner/llm.ts`.
   - consume `LLMClient.Service` de `@opencode-ai/llm` directamente;
   - trabaja con `LLM.request()` y `LLMEvent` portables.

El propio código de `packages/opencode/src/session/llm.ts` denomina AI SDK **“Default runtime path”**. Por tanto, `packages/llm` es arquitectura real y en uso, pero no debe documentarse como sustitución completa del runtime legacy-compatible.

## Arquitectura de `packages/llm`

`packages/llm` define un modelo intermedio normalizado y separa:

1. provider facade/configuración;
2. route: protocol + endpoint + auth + transport + defaults;
3. protocol adapter: request portable ↔ wire format/event stream;
4. runtime común: compilación, retry, errores, reasoning, tool events y usage.

Protocolos nativos presentes en `dev` incluyen:

- `openai-responses`;
- `openai-chat`;
- `openai-compatible-chat`;
- `anthropic-messages`;
- `gemini`;
- `bedrock-converse`.

Un provider no equivale a un protocolo: deployments distintos pueden reutilizar el mismo adapter protocolar.

## Product path: `packages/opencode/src/session/llm.ts`

### Preparación

`LLM.Service` recibe:

- `SessionV1.User`;
- Session/parent IDs;
- `Provider.Model`;
- `Agent.Info`;
- permisos;
- system/messages;
- tools;
- tool choice/retries.

Resuelve en paralelo:

- lenguaje/modelo ejecutable desde `Provider.Service`;
- configuración;
- provider metadata/config;
- auth.

`LLMRequestPrep.prepare()` aplica preparación común antes de elegir runtime.

### Runtime nativo opt-in

Con `flags.experimentalNativeLlm`:

```text
LLMNativeRuntime.stream(...)
  -> supported: usar @opencode-ai/llm
  -> unsupported: registrar razón y fallback a AI SDK
```

Por tanto, el native path en esta superficie es un **seam experimental/gradual**, no la única implementación.

### Runtime por defecto

Si no se selecciona el nativo, se llama a `streamText()` de AI SDK con:

- model wrapping/provider transforms;
- messages preparados;
- tools/activeTools;
- toolChoice;
- maxOutputTokens;
- abort signal;
- headers;
- telemetry;
- reparación de tool calls inválidas.

La reparación intenta corregir casing de tools y, si no puede, redirige la llamada a la tool `invalid`.

### GitLab workflow models

Existe además tratamiento específico para `GitLabWorkflowLanguageModel`, incluyendo ejecución de tools y aprobación de server-side workflow tools. Esto confirma que `Provider`/`LLM` de `packages/opencode` sigue siendo un composition boundary con compatibilidad provider-specific importante.

## Core V2: `@opencode-ai/llm`

En el runner V2, `LLMClient.Service` ejecuta directamente requests portables. `packages/core/src/session/runner/model.ts` resuelve actualmente de forma nativa sólo un subconjunto del catálogo heredado:

- `@ai-sdk/openai` → ruta OpenAI nativa;
- `@ai-sdk/anthropic` → Anthropic Messages;
- `@ai-sdk/openai-compatible` con URL explícita → OpenAI-compatible Chat.

`packages/llm` soporta más providers/protocols que ese resolver V2.

## Request preparation y streaming nativos

`packages/llm/src/route/client.ts` compone route/defaults/request. `compile` aplica policy de cache, lowering protocolar y validación antes de I/O.

El vocabulario común incluye eventos de:

- step;
- text;
- reasoning;
- tool input/call/result/error;
- finish;
- provider error.

Hosted/server-side tools se distinguen mediante `providerExecuted` cuando el protocolo lo permite.

## Reasoning y continuidad

Los adapters conservan metadata opaque necesaria para continuidad cuando aplica:

- OpenAI reasoning/item state;
- Anthropic thinking signatures;
- Gemini thought signatures;
- Bedrock reasoning signatures.

La portabilidad de esa metadata depende del mismo provider/model/protocol y no debe asumirse cross-provider.

## Retry, errores y caching

El runtime nativo posee una política central de retry/clasificación en `route/executor.ts`, independiente del retry de sesión/overflow.

`cache-policy.ts` normaliza markers/policies donde el protocolo los soporta. La estrategia exacta depende del adapter y no convierte el cache en una garantía universal de todos los providers.

## Usage y coste

`packages/llm` normaliza usage, incluido cache/reasoning cuando está disponible, pero no debe considerarse la autoridad de pricing. El coste se combina en capas de catálogo/core/session según el path.

En el runtime `packages/opencode`, `Session.getUsage()` también contiene compatibilidad provider-specific para metadata de cache y accounting; esto es otra señal de que la migración aún no ha concentrado toda la semántica en `packages/llm`.

## Lectura evolutiva

La hipótesis más consistente con el código es una migración por estrangulamiento:

```text
Provider/AI SDK legacy-compatible
       |
       +--> request/event seams
       +--> packages/llm protocol runtime
       +--> native runtime opt-in en SessionPrompt
       +--> Core V2 consume LLMClient directamente
       `--> retirada total de AI SDK aún no consumada en dev
```

## Índice

- [`branches/inventario-y-evolucion.md`](branches/inventario-y-evolucion.md)
- [`architecture/runtime-comun.md`](architecture/runtime-comun.md)
- [`protocols/openai-responses.md`](protocols/openai-responses.md)
- [`protocols/anthropic-gemini-bedrock.md`](protocols/anthropic-gemini-bedrock.md)
- [`providers/provider-specific.md`](providers/provider-specific.md)
- [`migration/native-stack-y-remove-ai-sdk.md`](migration/native-stack-y-remove-ai-sdk.md)

## Fuentes de mayor peso

### Runtime de producto

- `packages/opencode/src/session/llm.ts`
- `packages/opencode/src/session/llm/request.ts`
- `packages/opencode/src/session/llm/native-runtime.ts`
- `packages/opencode/src/provider/provider.ts`
- `packages/opencode/src/provider/transform.ts`

### Stack nativo/Core V2

- `packages/llm/DESIGN.md`
- `packages/llm/src/llm.ts`
- `packages/llm/src/provider.ts`
- `packages/llm/src/route/client.ts`
- `packages/llm/src/route/executor.ts`
- `packages/llm/src/cache-policy.ts`
- `packages/core/src/session/runner/model.ts`

## Conclusión

La afirmación correcta no es “OpenCode ya abandonó AI SDK” ni “packages/llm es sólo experimental”. `dev` contiene un runtime nativo maduro y un runner V2 que lo consume directamente, mientras el producto `packages/opencode` todavía usa AI SDK por defecto y selecciona el native path de forma opt-in con fallback. Esa coexistencia es el estado real que debe reflejar toda la documentación de providers.