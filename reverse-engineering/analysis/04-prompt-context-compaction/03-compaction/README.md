# Compaction y summaries

## Corrección de auditoría

`dev` contiene **dos implementaciones de compaction**. La versión anterior de este documento describía `packages/core/src/session/compaction.ts` como si fuese la única implementación vigente. Eso es incorrecto.

- El path de producto compuesto por `SessionPrompt` usa `packages/opencode/src/session/compaction.ts`.
- El runner V2/Core usa `packages/core/src/session/compaction.ts`.

Comparten ideas y algún helper, pero no tienen exactamente los mismos presupuestos, persistencia ni control flow.

---

## 1. Compaction del path de producto (`packages/opencode`)

Archivo principal: `packages/opencode/src/session/compaction.ts`.

### Selección de historia

La implementación agrupa la historia en turns iniciados por mensajes de usuario y selecciona una cola reciente que intenta caber en un presupuesto.

El presupuesto reciente por defecto es:

```text
25 % del contexto utilizable
clamp mínimo: 2.000 tokens
clamp máximo: 15.000 tokens
```

Puede sobrescribirse mediante `compaction.preserve_recent_tokens`. `compaction.tail_turns` puede limitar cuántos turns recientes se consideran.

Si un turn completo no cabe, `splitTurn()` intenta conservar una cola interna del propio turn.

### Contexto utilizable

La función `usable()` procede de `packages/opencode/src/session/overflow.ts`; usa los límites del modelo y la reserva configurada/output máximo. Por tanto, el presupuesto de recent no es un 8k fijo universal.

### Serialización para el resumen

La función `serialize()` incluye:

- texto de usuario y attachments como marcadores;
- texto assistant;
- reasoning;
- tool call + result/error.

Los outputs de tool incluidos en el material de resumen se truncan a **2.000 caracteres**; outputs que ya fueron podados aparecen como `[Old tool result content cleared]`.

### Summary incremental

La implementación detecta compactions previas completadas y recupera el `previousSummary`. Después usa `buildPrompt()` de `@opencode-ai/core/session/compaction` para integrar historia nueva con el resumen anterior.

Los plugins pueden modificar la compaction mediante `experimental.session.compacting` y `experimental.chat.messages.transform`.

### Agente/modelo

Se usa el agente interno `compaction`. Si éste fija modelo se usa ese modelo; en caso contrario se usa el modelo asociado al mensaje de usuario de compaction.

### Pruning de tools

Este mismo servicio implementa pruning independiente del summary:

- `PRUNE_PROTECT = 40_000` tokens recientes de tool output;
- sólo se poda si el ahorro supera `PRUNE_MINIMUM = 20_000`;
- `skill` está protegido por `PRUNE_PROTECTED_TOOLS`.

El pruning marca `part.state.time.compacted` en resultados antiguos en vez de confundirlo con la creación del resumen.

### Overflow compaction

El path legacy-compatible tiene lógica especial cuando `overflow` es true: puede retirar temporalmente el último user turn de la historia a resumir y reinyectarlo después para recuperar el intento. Esta semántica pertenece a `SessionPrompt`/`SessionCompaction` y no debe confundirse con el retry V2 de `SessionRunner`.

---

## 2. Compaction V2/Core

Archivo principal: `packages/core/src/session/compaction.ts`.

Esta implementación trabaja sobre el modelo durable V2, `SessionHistory`, `SessionMessage.Compaction` y requests de `@opencode-ai/llm`.

Entre sus propiedades confirmadas están:

- separación entre tramo resumible y recent context;
- summary estructurado mediante `buildPrompt()`;
- checkpoint V2 materializado posteriormente como contexto histórico;
- integración con `SessionContextEpoch`;
- auto-compaction antes de un provider turn en el runner V2;
- recovery específico después de context-overflow previo a salida assistant.

Los defaults concretos de esta implementación —por ejemplo buffers o límites del summary— deben atribuirse explícitamente a **V2/Core**, no al producto entero.

### Checkpoint y autoridad

`packages/core/src/session/runner/to-llm-message.ts` proyecta la compaction V2 como un mensaje de contexto histórico, evitando promover el resumen a instrucciones privilegiadas. Ésta es una propiedad del transcript V2.

---

## 3. Qué comparten ambas generaciones

Las dos implementaciones convergen en principios comunes:

1. resumir historia antigua;
2. conservar una cola reciente con mayor fidelidad;
3. reutilizar un summary anterior cuando existe;
4. reservar espacio para poder seguir generando;
5. no tratar outputs antiguos ilimitados como contexto gratuito.

Pero **principio común no implica implementación común**.

---

## 4. Branches históricas

### `compaction-study`, `bounded-compaction`, `bounded-compaction-prod`

Sirven como evidencia de la evolución de presupuestos y selección de cola. No deben utilizarse para sustituir los valores que aparecen realmente en las dos implementaciones de `dev`.

### `cache-friendly-compaction`

Exploró preservar prefijos para cache del proveedor. No es la política general de ninguna de las dos implementaciones vigentes observadas.

### `compaction-cleanup`

Aporta evidencia histórica sobre lifecycle/fallos y separación entre compaction manual, automática y recovery. Algunas preocupaciones sobreviven, pero las APIs vigentes son distintas.

### `compaction-steer`

Pertenece principalmente a scheduling/admission alrededor de compaction, no al formato del summary.

## Conclusión

La descripción correcta de `dev` es:

```text
SessionPrompt -> packages/opencode/src/session/compaction.ts
                selección por turns + recent budget dinámico + legacy/V1 projections

SessionRunner V2 -> packages/core/src/session/compaction.ts
                    durable V2 checkpoint + ContextEpoch + overflow recovery V2
```

Cualquier cifra o propiedad de compaction debe indicar qué implementación la posee.