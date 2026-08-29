# Context limits, overflow y retry

## Corrección de auditoría

`dev` tiene dos políticas de overflow asociadas a sus dos runtimes de sesión. La versión anterior de este documento describía el recovery del runner V2 como si fuese el comportamiento universal de OpenCode. La separación correcta es la siguiente.

---

## 1. Path de producto: `packages/opencode`

Archivos principales:

- `packages/opencode/src/session/overflow.ts`
- `packages/opencode/src/session/compaction.ts`
- `packages/opencode/src/session/prompt.ts`
- `packages/opencode/src/session/processor.ts`

### Contexto utilizable

`usable()` obtiene `model.limit.context`. Si el modelo declara `context === 0`, devuelve 0.

La reserva es:

```text
cfg.compaction.reserved
  ?? min(20_000, ProviderTransform.maxOutputTokens(model, outputTokenMax))
```

Si el modelo declara un límite de input separado:

```text
usable = max(0, model.limit.input - reserved)
```

si no:

```text
usable = max(0, model.limit.context - maxOutputTokens)
```

Es importante no confundir el constante `COMPACTION_BUFFER = 20_000` con una resta universal de 20k: por defecto se usa el **mínimo entre 20k y el output máximo calculado** como `reserved` cuando existe `model.limit.input`; en la rama sin input limit se resta directamente el output máximo.

### Detección de overflow/auto-compaction

`isOverflow()` devuelve `false` cuando:

- `cfg.compaction.auto === false`; o
- `model.limit.context === 0`.

El consumo usado para la comparación es:

```text
tokens.total
  || input + output + cache.read + cache.write
```

y se considera overflow cuando:

```text
count >= usable(...)
```

Este cálculo se basa en usage del assistant/modelo y no en la estimación completa de `request.system + messages + tools` utilizada por el runner V2.

### Recovery en el loop legacy-compatible

`SessionPrompt` y `SessionCompaction` poseen su propio control de continuidad. Cuando el processor solicita compaction, el loop crea el mensaje/part de compaction y continúa desde el historial resultante.

La implementación de `packages/opencode/src/session/compaction.ts` tiene además una ruta `overflow` capaz de separar/reproducir el último user turn para intentar recuperar una conversación que ya excedió contexto.

No debe describirse esta ruta con el mismo contrato de “interceptar `ProviderErrorEvent` antes de output y reintentar exactamente una vez” del runner V2: son implementaciones distintas.

---

## 2. Runner V2/Core

Archivos principales:

- `packages/core/src/session/runner/llm.ts`
- `packages/core/src/session/compaction.ts`

### Compaction proactiva

El runner V2 construye primero un `LLM.request()` portable y entrega request + historia + modelo a `compactIfNeeded()`. Esa implementación puede estimar el request ensamblado y decidir compactar **antes** del provider call.

Los límites y reservas concretos pertenecen a la implementación V2/Core y no deben proyectarse al `SessionPrompt` legacy-compatible.

### Overflow reactivo

Durante el stream V2 se reconoce `isContextOverflowFailure(...)` tanto en provider events como en failures normalizados.

El recovery sólo se intenta cuando:

```text
publisher.hasAssistantStarted() === false
```

Si ya comenzó salida assistant, el error no se oculta para repetir el turno desde cero.

### Recovery acotado

Cuando el primer attempt desborda antes de salida:

1. se retiene/clasifica el overflow;
2. se ejecuta `compactAfterOverflow(...)`;
3. si progresa, se produce `ContinueAfterOverflowCompaction`;
4. se reconstruye el request desde historia compactada;
5. el attempt siguiente se ejecuta sin volver a habilitar el mismo recovery.

Por tanto, **en el runner V2** el retry por overflow está acotado a una única reconstrucción/repetición.

Esto es independiente de retries HTTP/transporte del cliente LLM.

---

## 3. Diferencias que deben mantenerse explícitas

| Propiedad | `packages/opencode` | Core V2 |
|---|---|---|
| Trigger principal | usage/estado del loop y processor | estimación del request + provider overflow |
| `compaction.auto=false` | desactiva `isOverflow()` automático | la política depende de la función V2 invocada; recovery reactivo es separado |
| Representación | mensajes/parts SessionV1 compatibles | Session events/messages V2 |
| Recovery de overflow | lógica propia en `SessionCompaction.process(... overflow)` | `compactAfterOverflow` + `TurnTransitionError` |
| Retry “exactamente una vez” | no debe afirmarse como contrato general | sí, para el recovery reactivo V2 descrito |

## 4. Branches históricas

`feat/core-v2-overflow-recovery` es evidencia directa de la semántica que aparece en el **runner V2** actual. `context-overflow` ayuda a reconstruir la clasificación semántica del error. `prompt-retry` se refiere a admission/UI y no debe confundirse con retry LLM por context overflow.

## Conclusión

La documentación correcta debe evitar una política de overflow ficticiamente unificada. `dev` mantiene el algoritmo legacy-compatible de `SessionPrompt` y, a la vez, un recovery V2 más explícito basado en request portable, compaction durable y una transición de retry acotada.