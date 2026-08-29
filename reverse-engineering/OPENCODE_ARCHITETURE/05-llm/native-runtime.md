# Native LLM Runtime

**Status:** VERIFIED-CODE

## Purpose

El runtime nativo integra `@opencode-ai/llm` como alternativa experimental al AI SDK manteniendo el mismo stream `LLMEvent` consumido por `SessionProcessor`.

## Exact support gating

`LLMNativeRuntime.status()` exige simultáneamente:

1. provider ID `openai`, `anthropic` o prefijo `opencode`;
2. package de model API `@ai-sdk/openai`, `@ai-sdk/openai-compatible` o `@ai-sdk/anthropic`;
3. auth compatible; OAuth requiere provider fetch override en el caso soportado;
4. API key resoluble desde provider options/key.

Si alguna condición falla, devuelve `{ type: "unsupported", reason }`. `session/llm.ts` registra la razón y hace fallback explícito a AI SDK.

## Request mapping

La integración convierte model/provider metadata, messages, tools, tool choice, sampling, output limit, headers/auth y provider options a `LLMRequest`.

`ProviderTransform.message()` y `ProviderTransform.providerOptions()` siguen siendo parte del boundary; el comentario de implementación deja claro que los AI-SDK-shaped options y el SDK nativo deben referirse a los mismos wire fields oficiales.

## Tool bridge

Las tools OpenCode se convierten a `NativeTool`. **La ejecución sigue siendo propiedad de OpenCode**: el adapter invoca el `Tool.execute` del host con call ID, messages y abort signal. `ToolRuntime.dispatch` genera los eventos de result/error que se concatenan al stream nativo.

Las tool calls marcadas `providerExecuted` no se redispatchan localmente.

## Common event contract

- native path: ya produce `LLMEvent`;
- AI SDK path: `LLMAISDK.toLLMEvents()` convierte `fullStream` a `LLMEvent`.

Ese contrato común es el seam que desacopla `SessionProcessor` de los dos runtimes.

## Risk surface for dynamic validation

La auditoría estática no demuestra equivalencia temporal perfecta. Deben probarse dinámicamente:

- ordering de events;
- provider metadata/usage;
- tool serialization y provider-executed calls;
- reasoning blocks;
- stop/finish reasons;
- cache policy;
- cancel/retry timing.

## Sources

- `packages/opencode/src/session/llm/native-runtime.ts` — `bac385c59137ced710073051ed6388bc376e39ab`
- `packages/opencode/src/session/llm.ts` — `a99f8acff20c5d64d0b6cb90df480218bb1daddc`
- `packages/llm/src/llm.ts` — `e4781d8608b0185c500866aae20fda8335640550`
- `packages/llm/src/tool-runtime.ts` — `d69bbb9d478ca532a1481e8d7502c8e5c2b55dc6`
