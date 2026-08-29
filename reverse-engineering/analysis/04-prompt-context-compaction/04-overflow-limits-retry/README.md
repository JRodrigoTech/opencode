# Context limits, overflow y retry

## Modelo de límites en `dev`

El límite operativo de contexto procede del modelo resuelto:

```text
model.route.defaults.limits.context
```

El presupuesto de output se obtiene, en orden práctico, de:

```text
request.generation.maxTokens
model.route.defaults.limits.output
0
```

`SessionCompaction` no interpreta un número fijo global como context window. Trabaja con los límites de la ruta/modelo que realmente se va a usar.

## Reserva proactiva antes del overflow

**CONFIRMADO EN `dev`:** antes de ejecutar el provider turn, el runner entrega a `compactIfNeeded()` el request ya ensamblado. La estimación incluye:

- `request.system`;
- `request.messages`;
- `request.tools`.

Se compacta cuando la estimación excede:

```text
context - max(outputBudget, compaction.buffer)
```

con `compaction.buffer = 20_000` tokens por defecto.

Esto implica que OpenCode no espera necesariamente a que el proveedor rechace el request. Intenta mantener una reserva para:

- el output del modelo;
- variación/error de estimación;
- crecimiento durante tool loops/continuations.

### Si el context limit no está disponible

**CONFIRMADO EN `dev`:** la auto-compaction basada en tamaño devuelve `false` si el modelo no declara un `limits.context` positivo. No inventa una ventana.

## El propio summary también debe caber

Antes de enviar un summary request, `compactAfterOverflow()` calcula:

```text
summaryOutput = min(outputBudget || 4096, 4096)
```

y rechaza la operación si el `summaryPrompt` estimado supera:

```text
context - summaryOutput
```

**CONFIRMADO EN `dev`:** esto evita una recursión absurda en la que la operación destinada a resolver el overflow también desborda la ventana.

## Overflow reactivo del proveedor

Archivo principal: `packages/core/src/session/runner/llm.ts`.

### Detección

El stream detecta `isContextOverflowFailure(event)` para provider errors. Hay dos casos relevantes:

1. overflow reportado como `ProviderErrorEvent` dentro del stream;
2. overflow materializado como failure del stream/`LLMError`.

### Condición de seguridad

**CONFIRMADO EN `dev`:** el error sólo se intercepta para recovery si `publisher.hasAssistantStarted()` es falso.

Es decir: si ya comenzó a publicarse respuesta assistant, OpenCode no oculta el fallo para reejecutar el turno desde cero. Esta condición previene duplicar side effects o producir dos respuestas solapadas para una misma entrada.

## Recovery path

Cuando el overflow ocurre antes de salida assistant:

1. se retiene el error en `overflowFailure` en lugar de publicarlo inmediatamente;
2. termina/se evalúa el provider stream;
3. se invoca `compactAfterOverflow({ sessionID, entries, model, request })`;
4. si la compaction termina correctamente, el runner dispara `ContinueAfterOverflowCompaction`;
5. el turno se reconstruye desde historia persistida compactada;
6. el nuevo attempt se ejecuta **sin volver a proporcionar la función de overflow recovery**.

El paso 6 es el mecanismo que acota el retry.

## Retry exactamente una vez

**CONFIRMADO EN `dev`:** la arquitectura no implementa `while overflow -> compact -> retry` indefinido. La primera ejecución dispone de `recoverOverflow`; el attempt posterior a la compaction se ejecuta por el camino que ya no vuelve a recuperar otro overflow.

Por tanto la semántica efectiva es:

```text
provider attempt
  -> overflow antes de assistant output
  -> compact
  -> rebuild request
  -> one retry
  -> si vuelve a fallar, propagar fallo
```

## Diferencia entre auto-compaction y overflow recovery

Son dos triggers distintos:

| Trigger | Momento | Requiere `compaction.auto` | Propósito |
|---|---|---:|---|
| `compactIfNeeded` | antes del provider call | sí | prevención |
| `compactAfterOverflow` | después de context-overflow | no depende del gate de auto | recuperación |

**CONFIRMADO EN `dev`:** aunque `compactIfNeeded()` comprueba `config.auto`, `compactAfterOverflow()` no lo hace. Por diseño, desactivar auto-compaction no elimina necesariamente la capacidad de salvar un turno que el proveedor ya rechazó por overflow.

## Branch `feat/core-v2-overflow-recovery`

**CONFIRMADO EN BRANCH:** esta rama introduce de manera explícita el patrón de recovery que hoy puede reconocerse en `dev`: capturar overflow previo a salida, compactar y reintentar una vez reconstruyendo el request.

**EVOLUCIÓN:** la implementación actual usa `TurnTransitionError` y dos transiciones distintas:

- `ContinueAfterCompaction` para compaction proactiva;
- `ContinueAfterOverflowCompaction` para la recuperación reactiva.

Esto hace que la política de retry esté codificada en el control flow del runner y no en un retry genérico del provider client.

## `context-overflow`

**CONFIRMADO EN BRANCH:** la rama estudia la clasificación/propagación del overflow como una condición semántica distinta de un error de transporte o error genérico del proveedor.

**CONFIRMADO EN `dev`:** esa idea queda formalizada mediante `isContextOverflowFailure()` y el tratamiento especial en el runner.

## `prompt-retry` no es este mecanismo

La branch `prompt-retry` contiene en su punta `fix(tui): retry ambiguous prompt admission` (`d6affc0685e7daca8443a193a4957f0b2153af92`).

**CONFIRMADO:** se refiere a admission/retry del prompt en la TUI, no al retry de un LLM request tras context overflow. Se incluye en el inventario para evitar una asociación falsa por nombre.

## `remove-200k-context` tampoco define la ventana

La rama contiene `refactor(opencode): remove legacy 200k pricing` (`96135d70a4cb3fbcbe98d90f40c8762a50d30af3`) y cambios de tiers/pricing.

**CONFIRMADO:** el “200k context” de ese nombre está ligado principalmente a contratos de pricing legacy, no al algoritmo actual que decide cuándo compactar. No se usa como evidencia de context limit operativo.

## Fallos de compaction durante recovery

**CONFIRMADO EN `dev`:** `compactAfterOverflow()` puede devolver `false` si:

- no hay context limit utilizable;
- no existe historial seleccionable;
- no hay `head` antiguo y tampoco checkpoint previo que permita progresar;
- el summary prompt no cabe;
- el summarizer emite provider error;
- el stream del summarizer falla;
- el resultado queda vacío.

En esos casos no se genera la transición de recovery y el overflow original termina siendo publicado/propagado.

## Qué no hace este mecanismo

- No reduce dinámicamente tools individuales para intentar encajar.
- No baja automáticamente a otro modelo con ventana mayor.
- No implementa múltiples niveles de summary recursivo en `dev`.
- No trata un overflow posterior a salida assistant como retry seguro.
- No confunde retry por overflow con los retries de transporte/provider, que pertenecen a otra capa.

## Conclusión

**INFERENCIA, confianza alta:** la política está diseñada como una máquina de estados, no como un retry middleware. La compaction cambia durablemente la representación de la sesión; por eso el retry correcto requiere **reconstruir el request desde la nueva historia**, no reenviar el mismo payload. Esta es la razón arquitectónica por la que overflow recovery reside en `SessionRunner` y no exclusivamente en el adapter del proveedor.