# Provider Transformation Layer

**Status:** VERIFIED-CODE + DERIVED

## Role

`packages/opencode/src/provider/transform.ts` es el compatibility layer entre el modelo lógico de OpenCode/AI SDK y las expectativas concretas de modelos/proveedores.

Debe leerse junto con:

- `provider/provider.ts`
- `session/llm/request.ts`
- `session/llm.ts`

## Transformation domains

La capa transforma al menos:

- messages/prompt parts;
- JSON schemas de tools;
- provider options;
- model-specific quirks;
- reasoning/cache metadata;
- IDs/names donde proveedores imponen restricciones.

## Why it exists

La lista de providers es demasiado heterogénea para asumir equivalencia de wire format. Algunos providers usan OpenAI-compatible endpoints, otros SDKs específicos, otros necesitan routing por modelo a chat/responses/messages.

## Schema transform

`SessionTools.resolve` aplica `ProviderTransform.schema(input.model, tool.jsonSchema)` antes de exponer tools al LLM. Por tanto, el schema que ve el modelo puede no ser byte-identical al schema del registry.

## Message transform

El AI SDK middleware invoca `ProviderTransform.message` durante `transformParams`. El runtime nativo aplica la misma familia de transformaciones antes de construir request.

## Audit rule

Cuando una conducta aparece “solo con un modelo”, el orden de inspección recomendado es:

1. `Provider.getLanguage`
2. `LLMRequestPrep.prepare`
3. `ProviderTransform.message/schema`
4. SDK/provider package
5. event conversion.

## Sources

- `packages/opencode/src/provider/transform.ts` — `28a5beb9abacdf1546d9c3a4492b25e0e917f062`
- `packages/opencode/src/provider/provider.ts` — `b5980f15...`
