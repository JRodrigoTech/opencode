# Adaptaciones provider-specific

## Principio general

**[HECHO]** En OpenCode, provider y protocol son conceptos distintos. La facade de provider resuelve autenticación, endpoint, defaults y selección de route; el adapter de protocolo traduce mensajes/eventos.

Este diseño permite reutilizar un protocolo en deployments diferentes sin duplicar el parser.

---

# OpenAI

Provider: `packages/llm/src/providers/openai.ts`.

**[HECHO]** Routes disponibles en `dev`:

- OpenAI Responses HTTP;
- OpenAI Responses WebSocket;
- OpenAI Chat.

**[HECHO]** `model()` selecciona Responses por defecto.

**[HECHO]** Auth bearer desde opción explícita o `OPENAI_API_KEY`.

**[HECHO]** La facade añade options tipadas OpenAI sobre un protocol adapter reusable.

## Branches relevantes

- `openai-websocket`
- `openai-text-phase`
- `fix/openai-stateless-reasoning-continuation`
- `fix/openai-codex-route-timeouts`
- `fix/openai-websocket-header-timeout`
- `llm-native-runtime-openai`

**[INFERENCIA]** OpenAI fue el provider piloto del runtime nativo; después, sus seams se generalizaron a otros protocolos.

---

# Azure OpenAI

Provider: `packages/llm/src/providers/azure.ts`.

## Reutilización de protocolo

**[HECHO]** Azure no implementa su propio protocol parser. Define routes derivadas de:

- OpenAI Responses;
- OpenAI Chat.

pero cambia provider id, auth y endpoint.

## Endpoint

**[HECHO]** Con `resourceName`, la URL base es:

```text
https://<resource>.openai.azure.com/openai/v1
```

También se puede suministrar `baseURL` directamente.

**[HECHO]** Se inyecta `api-version`, con `v1` como default de las routes del baseline.

## Auth

**[HECHO]** El baseline elimina un `authorization` heredado y usa header `api-key`, con fallback `AZURE_OPENAI_API_KEY`.

## Chat vs Responses

**[HECHO]** `useCompletionUrls === true` selecciona Chat; en caso contrario `model()` usa Responses.

Branches:

- `azure-chat-reasoning`
- `azure-reasoning-cleanup`
- `azure-cognitive-improvements`
- `azure-cli-auth`
- `fix-azure-issue`

**[HISTÓRICO]** La matriz de `remove-ai-sdk` considera Azure “partial”: existía facade nativa pero faltaban mapping de catálogo completo, AAD/token auth y validación de variantes de endpoint.

**[INFERENCIA]** Azure demuestra que protocol compatibility y auth compatibility son dimensiones separadas.

---

# Google Gemini

Provider: `packages/llm/src/providers/google.ts`.

**[HECHO]** Usa el protocolo Gemini nativo.

Auth:

- option apiKey;
- fallback `GOOGLE_GENERATIVE_AI_API_KEY`;
- header `x-goog-api-key`.

**[HECHO]** Esta facade corresponde a Gemini Developer API, no a Vertex.

---

# Google Vertex

La evolución posterior de `packages/ai` divide Vertex en varias facades.

**[HISTÓRICO]** La matriz `remove-ai-sdk` documenta cuatro slices:

1. Vertex Gemini;
2. Vertex Chat;
3. Vertex Responses;
4. Vertex Messages.

Esto implica cuatro wire protocols bajo una plataforma/provider de deployment.

## Vertex Gemini

**[HECHO]** `vertex-multiregion` contiene una facade que:

- acepta project/location;
- deriva hostname regional;
- acepta API key Express Mode u OAuth;
- soporta tuned model IDs `endpoints/...`;
- prohíbe API key Express Mode para tuned endpoints;
- conserva labels provider-specific.

## Vertex Chat

**[HISTÓRICO]** Usado para MaaS models expuestos como OpenAI-compatible Chat.

## Vertex Responses

**[HISTÓRICO]** Usado para Grok/Open Responses deployments. La matriz de paridad señala `store:false` como default Vertex y ausencia de stateful continuation.

## Vertex Messages

**[HISTÓRICO]** Anthropic Messages sobre Vertex, con OAuth/ADC y endpoints globales/regionales/multi-region.

Branches:

- `vertex-fixes`
- `vertex-multiregion`
- `vertex-xai-default`
- `fix/google-vertex-anthropic-auth-message`
- `fix/google-vertex-anthropic-thinking`
- `fix/google-vertex-metadata-warning`

**[INFERENCIA]** Vertex es la evidencia más fuerte de que el resolver debe elegir route a partir de deployment API, no únicamente a partir del fabricante nominal del modelo.

---

# Amazon Bedrock

Provider baseline: `packages/llm/src/providers/amazon-bedrock.ts`.

**[HECHO]** Usa Bedrock Converse.

**[HECHO]** Endpoint calculado por región:

```text
https://bedrock-runtime.<region>.amazonaws.com
```

Default regional baseline: `us-east-1` si no se especifica otro.

**[HECHO]** Auth:

- bearer si hay API key;
- SigV4 en caso contrario.

## Evolución auth

- `bedrock-region`
- `bedrock-credentials`
- `bedrock-credential-chain`
- `bedrock-metadata-terminal`

**[HECHO]** La credential-chain posterior no se mete en la route: se resuelve en core y se entrega al signer.

## Bedrock Mantle

**[HISTÓRICO]** La generación `remove-ai-sdk` añade una familia Mantle con Chat y Responses OpenAI-compatible además de Converse.

**[INFERENCIA]** Igual que Vertex, Bedrock puede convertirse en un hosting provider multiprotocolo; “amazon-bedrock” no determina necesariamente un único wire format a futuro.

---

# xAI

Provider baseline: `packages/llm/src/providers/xai.ts`.

**[HECHO]** Reutiliza:

- OpenAI Responses;
- OpenAI-compatible Chat.

**[HECHO]** La URL por defecto procede de un OpenAI-compatible profile para xAI y la auth usa bearer con `XAI_API_KEY`.

**[HECHO]** Responses es el default de `model()`.

Branches:

- `xai-capacity-text`
- `xai-capacity-v2`
- `xai-device-default`
- `xai-oauth`
- `fix/xai-provider-upstream-pdf-support`
- `vertex-xai-default`

**[INFERENCIA]** Las branches de xAI mezclan tres planos que el diseño intenta mantener separados: capacidad/model metadata, autenticación y protocolo/deployment.

---

# DeepSeek

Branches:

- `deepseek-anthropic`
- `deepseek-responses`
- `deepseek-weekends`
- `fix-deepseek-reasoner`

**[HECHO]** En el baseline existe perfil DeepSeek sobre OpenAI-compatible Chat.

**[HISTÓRICO]** La matriz posterior incluye DeepSeek entre los perfiles nativos de OpenAI-compatible Chat.

**[INFERENCIA]** `deepseek-responses` y `deepseek-anthropic` prueban que el nombre del vendor no define por sí solo el protocolo: gateways/deployments pueden presentar al mismo modelo bajo dialectos distintos.

**[INFERENCIA]** `fix-deepseek-reasoner` señala una segunda fuente de incompatibilidad: incluso usando wire shape OpenAI-like, el reasoning puede requerir transformaciones específicas.

---

# OpenAI-compatible providers

La generación posterior documenta perfiles de Chat para, entre otros:

- Baseten;
- Cerebras;
- DeepInfra;
- DeepSeek;
- Fireworks;
- Groq;
- TogetherAI.

Además hay facades para OpenRouter y Cloudflare.

## Regla de diseño

**[HECHO]** OpenAI-compatible Chat y Open Responses-compatible se mantienen como protocolos separados.

**[HECHO]** El adapter Open Responses-compatible no hereda automáticamente tools/defaults/metadata exclusivas de OpenAI.

**[INFERENCIA]** La compatibilidad se trata como “dialecto base + profile”, no como licencia para enviar cualquier option OpenAI a cualquier endpoint compatible.

---

# Provider-specific options

**[HECHO]** El runtime portable permite `providerOptions` como namespace de escape hatch.

Ejemplos observados:

- OpenAI: reasoning effort, store, prompt cache key, include, service tier, verbosity;
- Gemini: thinking config;
- Vertex: labels y deployment settings en generaciones posteriores;
- Bedrock: `additionalModelRequestFields` y settings de auth/region;
- OpenRouter: opciones propias en la generación ampliada.

**[INFERENCIA]** La API común pretende cubrir semántica compartida; no intenta estandarizar prematuramente todas las opciones experimentales de cada vendor.

---

# Model resolution y routing

## Baseline `dev`

**[HECHO]** `SessionRunnerModel.fromCatalogModel` todavía selecciona sólo tres families de `aisdk` metadata.

## Evolución posterior

Branches `native-provider-*`, `ai-native-routing`, `remove-ai-sdk` y `bedrock-credential-chain` apuntan a un `ModelResolver` que:

1. resuelve catalog model + variant;
2. resuelve Integration/Credential;
3. mapea package heredado a provider package nativo cuando existe;
4. carga el provider package;
5. construye un `LanguageModel` route-level;
6. conserva por separado catalog `ref`, capabilities y pricing.

**[INFERENCIA]** Esta arquitectura convierte el routing en un compile step de configuración, no en lógica dispersa dentro del agent loop.

---

# Matriz conceptual

| Provider/deployment | Protocol(s) observados | Auth principal | Observación |
|---|---|---|---|
| OpenAI | Responses, Chat | Bearer | Responses default |
| Azure OpenAI | Responses, Chat | `api-key` | mismo protocol, endpoint/auth distintos |
| Anthropic | Messages | `x-api-key` | thinking signatures/server tools |
| Google Developer | Gemini | `x-goog-api-key` | no es Vertex |
| Google Vertex | Gemini, Chat, Responses, Messages | OAuth/ADC o API key según slice | multiprotocolo |
| Bedrock | Converse; Mantle Chat/Responses posterior | SigV4/bearer | AWS EventStream en Converse |
| xAI | Responses, Chat | Bearer | también alojable vía Vertex |
| DeepSeek | OpenAI-compatible; experimentos otros dialectos | deployment-specific | reasoning quirks |

## Conclusión

**[INFERENCIA]** El boundary más estable de OpenCode no es “Provider interface”, sino la combinación:

```text
Catalog model
  + Integration/Credential
  + Provider facade/package
  + Route
  + Protocol
  + Transport
```

Separar esos ejes permite añadir un nuevo deployment sin duplicar un protocolo entero y permite añadir un nuevo protocolo sin reescribir catálogo, sesión o UI.
