# Provider Matrix

**Status:** VERIFIED-CODE at provider registry level

La baseline integra adapters/SDKs para un conjunto amplio de providers. La matriz siguiente describe el boundary, no garantiza que cada modelo concreto esté habilitado en toda configuración.

| Family / Provider integration | Mechanism observed |
|---|---|
| Anthropic | `@ai-sdk/anthropic` + native path support |
| OpenAI | `@ai-sdk/openai`, Responses/compat routing, native support |
| OpenAI-compatible | `@ai-sdk/openai-compatible` / custom configs |
| Azure | `@ai-sdk/azure`, provider-specific endpoint selection |
| Amazon Bedrock | `@ai-sdk/amazon-bedrock` |
| Google Gemini | `@ai-sdk/google` |
| Google Vertex | `@ai-sdk/google-vertex` |
| OpenRouter | `@openrouter/ai-sdk-provider` |
| xAI | `@ai-sdk/xai` |
| Mistral | `@ai-sdk/mistral` |
| Groq | `@ai-sdk/groq` |
| DeepInfra | `@ai-sdk/deepinfra` |
| Cerebras | `@ai-sdk/cerebras` |
| Cohere | `@ai-sdk/cohere` |
| Together | `@ai-sdk/togetherai` |
| Perplexity | `@ai-sdk/perplexity` |
| Vercel / AI Gateway | Vercel/Gateway SDKs |
| Alibaba | `@ai-sdk/alibaba` |
| GitLab | `gitlab-ai-provider` + workflow special handling |
| GitHub Copilot | plugin/provider integration + model routing |
| Venice | `venice-ai-sdk-provider` |

## Selection complexity

`provider.ts` no es solo una map de nombres. Gestiona:

- credentials/env/config;
- model metadata;
- dynamic provider packages;
- base URLs;
- timeouts;
- endpoint flavor por modelo;
- provider-specific options.

Ejemplo conceptual: una integración compatible con OpenAI puede elegir entre chat-completions y responses según metadata/model, y Copilot/Azure tienen lógica adicional.

## Sources

- `packages/opencode/src/provider/provider.ts` — `b5980f15873b22647b03aa75fe450e2344aed5b9`
- `packages/opencode/package.json` — provider dependencies de production `1.18.25`
