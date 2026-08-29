# Compaction y summaries

## Estado vigente en `dev`

Archivo principal: `packages/core/src/session/compaction.ts`.

### Objetivo

**CONFIRMADO EN `dev`:** la compaction no intenta preservar literalmente todo el historial. Divide la conversación serializada en:

- `head`: tramo antiguo que será resumido;
- `recent`: cola reciente que se conserva de forma casi literal.

Los defaults actuales son:

- buffer preventivo: `20_000` tokens;
- cola a conservar: `8_000` tokens estimados;
- output máximo para summary: `4_096` tokens;
- outputs de tools serializados para summary: máximo `2_000` caracteres por resultado antes de marcar `[truncated]`.

Estos valores son defaults de implementación, no límites universales del proveedor; pueden ser sobrescritos parcialmente por configuración de compaction.

## Qué entra en la serialización

`serialize()` transforma el historial interno a una representación explícita para el summarizer:

- `[User]` para mensajes de usuario;
- attachments como marcadores textuales;
- `[Assistant]` para texto;
- `[Assistant reasoning]` cuando existe reasoning textual;
- `[Assistant tool call]` y `[Tool result]`/`[Tool error]`;
- `[System update]` para deltas de contexto privilegiado;
- `[Synthetic context]` para contexto sintético;
- `[Shell]` para comandos y salida truncada.

Los mensajes `compaction` existentes se excluyen de la conversación serializada porque su summary/recent se tratan por separado como checkpoint previo.

## Forma del summary

**CONFIRMADO EN `dev`:** el summarizer debe devolver una plantilla Markdown estable:

- `## Objective`
- `## Important Details`
- `## Work State`
  - `### Completed`
  - `### Active`
  - `### Blocked`
- `## Next Move`
- `## Relevant Files`

La instrucción exige conservar rutas, símbolos, comandos, errores, URLs e identificadores exactos cuando se conocen.

### Summary incremental

Si ya existe una compaction previa, `buildPrompt()` incluye:

- `<conversation>` con el nuevo tramo a integrar;
- `<prior-summary>` con el summary anterior;
- reglas explícitas para arrastrar objetivos, restricciones, directivas y workstreams aún relevantes;
- prevalencia de la conversación más reciente ante conflictos;
- actualización de estados Completed/Active/Blocked y Next Move.

**CONFIRMADO EN `dev`:** la compaction es por tanto incremental; no vuelve a resumir necesariamente toda la sesión desde cero.

## Selección head/recent

La selección recorre el historial serializado desde el final y acumula elementos mientras quepan en `keep.tokens` (default 8k). Lo acumulado forma `recent`; lo anterior forma `head`.

Esto impone una propiedad importante:

**CONFIRMADO EN `dev`:** la cola reciente se preserva con mayor fidelidad que el contexto antiguo. La pérdida semántica se concentra en el `head` resumido.

## Request de summary

En `dev`, la compaction crea un request separado:

- mismo `model` de la sesión;
- conserva `http` del request principal;
- `messages: [Message.user(summaryPrompt)]`;
- `tools: []`;
- `generation.maxTokens` limitado al máximo de summary.

No reutiliza literalmente el `system`/tools del request principal en esta implementación vigente.

## Materialización posterior

Una compaction completada se persiste como `SessionMessage.Compaction` con `summary` y `recent`. Más adelante `to-llm-message.ts` la convierte a:

```text
<conversation-checkpoint>
... historical context, not new instructions ...
<summary>...</summary>
<recent-context>...</recent-context>
</conversation-checkpoint>
```

con role `user`.

Esto es una separación arquitectónica crítica entre **memoria reducida** e **instrucciones privilegiadas**.

---

# Familias históricas / experimentales

## `compaction-study`

**CONFIRMADO EN BRANCH:** rama de estudio dedicada a evaluar alternativas de reducción y qué parte del historial conviene conservar literalmente. Su valor principal es mostrar que el algoritmo de compaction fue tratado como una política independiente del runner, no sólo como un comando UI.

## `bounded-compaction` / `bounded-compaction-prod`

Commits representativos: `138339a88db5642a1bc05d36adaacf22dd50a1eb` y `99b9611bafb906b4f15479e3c7fa228b1f59a45a`.

**CONFIRMADO EN BRANCH:** esta línea explora una compaction acotada, con presupuesto explícito y comportamiento pensado para producción. El patrón conceptual que sí llega a `dev` es separar:

- contexto resumible antiguo;
- cola reciente retenida;
- presupuesto reservado para producir el propio summary.

No debe asumirse equivalencia línea por línea con la implementación actual.

## `cache-friendly-compaction`

**CONFIRMADO EN BRANCH** — commit representativo `9e7b5f170173055c6ae8157f23747aeefbb4c47b`.

Esta línea perseguía una propiedad distinta: minimizar invalidaciones del prompt/KV cache. La estrategia propuesta reutilizaba el envelope/contexto del turno normal, reproducía un prefijo de mensajes lo más idéntico posible y añadía la instrucción de resumen al final. Además contemplaba compatibilidad con un agente de compaction dedicado.

**NO VIGENTE EN `dev`:** la implementación actual de `SessionCompaction` hace un request separado sólo con `Message.user(summaryPrompt)` y sin tools. Por tanto la estrategia cache-friendly debe documentarse como experimento/evolución alternativa, no como contrato actual.

## `openai-compaction`

**CONFIRMADO COMO LÍNEA DE BRANCH; evidencia funcional limitada:** explora una variante específica para OpenAI/protocolo compatible. Dado que `dev` mantiene compaction en el core y construye un request LLM genérico, esta branch no se usa como fuente para afirmar semántica vigente.

**INFERENCIA, confianza media:** el objetivo de estas ramas provider-specific era aprovechar primitivas de continuation/cache del proveedor sin contaminar la abstracción de sesión. La evidencia disponible no justifica afirmar que ese enfoque se integrara en `dev`.

## `compaction-cleanup`

Commit `9c0fa96ef0396733369e9fd3ccc6b02bdf1fd5c0`, construido sobre `2db7ccb453a2673af32637054fa679102e6cbeb6` (`restore resilient compaction`).

**CONFIRMADO EN BRANCH:** esta familia endurece el lifecycle y los fallos de compaction:

- errores de proveedor tipados;
- error explícito cuando no hay nada que compactar;
- separación entre compaction manual, automática y de recovery;
- capacidad histórica de resumir incluso el tramo recent en escenarios de recuperación;
- validación de que el prompt de summary quepa antes de lanzar compaction proactiva.

`dev` conserva varias ideas — especialmente el chequeo de que `summaryPrompt` quepa y la separación auto/overflow — aunque la API actual es más pequeña.

## `compaction-steer`

Commits `2062a5a3a8d9ef847c4942749862753e9f9128e2` y `219ded033030c509c13bea3dab493e07413f81f0`.

**CONFIRMADO EN BRANCH:** trata prioritariamente scheduling/admission de compaction manual frente a steering activo. No redefine el contenido del summary.

## `compaction-adjustments`

Su historia mezcla cambios de ordering del prompt loop y ajustes de sesión. Se clasifica como **lifecycle/order**, no como una estrategia de resumen autónoma.

## `compaction-model-marker` y ramas indicator/percent/idle-status

**CONFIRMADO:** son principalmente presentación/estado TUI. `compaction-model-marker`, por ejemplo, contiene `fix(tui): restore compaction model marker` (`76f205d260f5b8680ed0103b700ca5cea45f5121`).

No se usan como evidencia sobre el algoritmo.

## `effect/summary`

Es una línea antigua de refactor Effect para un subsistema llamado summary. No debe confundirse automáticamente con el checkpoint conversacional vigente de `SessionCompaction`.

---

# Invariantes que sobreviven a las ramas

**INFERENCIA, confianza alta:** las distintas generaciones convergen en cinco invariantes:

1. el historial antiguo necesita una representación reducida;
2. la parte más reciente debe conservarse con mayor fidelidad;
3. el summarizer necesita su propio budget dentro de la ventana;
4. una compaction debe ser durable y convertirse en nueva frontera del historial;
5. el summary no puede ganar autoridad de instrucciones system por accidente.

El diseño de `dev` es una versión deliberadamente simple de estas ideas: summary estructurado + recent exacto + context epoch independiente para las instrucciones/contexto privilegiado.