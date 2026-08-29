# LLM Abstraction

**Status:** VERIFIED-CODE

## Contract

`LLM.stream(input)` retorna `Stream<LLMEvent>`. Este es el boundary que desacopla `SessionProcessor` del runtime concreto.

## Preparation

Antes de iniciar stream se obtienen en paralelo:

- language model del provider;
- provider object/config;
- global config;
- provider auth.

Después `LLMRequestPrep.prepare` produce los parámetros efectivos: system, messages, tools, toolChoice, sampling, headers/providerOptions y otras opciones.

## Runtime choice

OpenCode soporta dos paths:

```text
LLM.stream
   │
   ├─ default/compatibility -> Vercel AI SDK `streamText`
   │
   └─ experimental/native -> @opencode-ai/llm
```

El runtime nativo se usa solo cuando flags y soporte de provider/model lo permiten.

## Normalized event vocabulary

Ambos paths producen conceptos equivalentes como:

- text start/delta/end
- reasoning start/delta/end
- tool input/call/result/error
- step start/finish
- provider error
- finish

Por eso `SessionProcessor` no contiene branches grandes por provider runtime.

## Tool ownership

Incluso en runtime nativo, las definitions se convierten para `@opencode-ai/llm`, pero la ejecución sigue integrándose con el tool runtime de OpenCode. El transporte LLM no obtiene ownership del workspace/policy.

## GitLab workflow special case

Existe tratamiento específico para modelos/workflows de GitLab que pueden ejecutar herramientas/aprobaciones de una forma distinta; se adapta igualmente al stream que espera el processor.

## Sources

- `packages/opencode/src/session/llm.ts` — `a99f8acff20c5d64d0b6cb90df480218bb1daddc`
- `packages/llm/src/llm.ts` — `e4781d8608b0185c500866aae20fda8335640550`
