# Native LLM Runtime

**Status:** VERIFIED-CODE

## Purpose

El runtime nativo integra `@opencode-ai/llm` como alternativa al AI SDK manteniendo la interfaz `LLMEvent` del host.

## Support gating

La integración inspecciona provider package/API y limita el uso nativo a combinaciones soportadas. En la baseline se reconoce soporte para familias OpenAI/OpenAI-compatible y Anthropic, incluyendo provider IDs `opencode*` bajo condiciones definidas.

## Request mapping

Convierte:

- model/provider metadata;
- system/messages;
- tools;
- tool choice;
- sampling/options;
- headers/auth;

al `LLMRequest` del package nativo.

## Tool bridge

Las tools OpenCode se convierten a definitions de `@opencode-ai/llm`. El dispatcher conecta las tool calls nativas de vuelta al executor del host y reinyecta tool result/error events en el stream.

## Compatibility intent

El código documenta explícitamente que provider options con shape de AI SDK y el SDK nativo deben mapear a los mismos official wire fields. El objetivo es reducir divergencia semántica durante la migración.

## Risk surface

La principal zona a validar dinámicamente al comparar ambos runtimes es:

- ordering de events;
- metadata/provider usage;
- tool argument/result serialization;
- reasoning blocks;
- stop/finish reasons;
- cache policy;
- request option equivalence.

## Sources

- `packages/opencode/src/session/llm/native-runtime.ts`
- `packages/llm/src/llm.ts`
- `packages/llm/src/tool-runtime.ts`
