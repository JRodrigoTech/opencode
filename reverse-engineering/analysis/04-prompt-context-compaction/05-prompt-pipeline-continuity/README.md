# Prompt pipeline, metadata y continuidad entre turns

## Construcción vigente del request

Archivo principal: `packages/core/src/session/runner/llm.ts`.

El runner ya no contiene un `SessionPrompt` monolítico. Su propia cabecera deja explícita la intención de mantenerlo como orquestación sobre colaboradores pequeños.

### Secuencia de un provider turn

**CONFIRMADO EN `dev`:** de forma simplificada, cada attempt realiza:

1. cargar la sesión y validar su location/workspace;
2. seleccionar el agente;
3. cargar/combinar system context, skill guidance y reference guidance;
4. inicializar o preparar `SessionContextEpoch`;
5. resolver el modelo;
6. cargar `SessionHistory.entriesForRunner()` usando `baselineSeq`;
7. materializar las tools permitidas para el agente;
8. construir `LLM.request()`;
9. intentar auto-compaction;
10. capturar snapshot del filesystem;
11. ejecutar `llm.stream(request)`;
12. persistir texto, reasoning, provider metadata, tool calls/results y usage durante el stream;
13. continuar si hay tools, steering u otra transición explícita.

## `request.system`

**CONFIRMADO EN `dev`:** se construye exactamente a partir de:

```text
[agent.info?.system, system.baseline]
```

filtrando valores vacíos y convirtiéndolos a `SystemPart`.

Esto confirma que:

- el system prompt del agente es independiente del contexto ambiental;
- el baseline contextual tiene también role system;
- los updates posteriores al baseline no tienen que volver a concatenarse aquí porque viven cronológicamente en el historial.

## `request.messages`

Se forma con:

```text
toLLMMessages(context, model)
```

y, si se alcanza el máximo de steps del agente, se añade un mensaje assistant `MAX_STEPS_PROMPT` y se deshabilitan tools (`toolChoice: "none"`).

## Proyección de cada tipo de mensaje

Archivo: `packages/core/src/session/runner/to-llm-message.ts`.

### User

Se conserva:

- `id`;
- texto;
- attachments como partes `media` con `mediaType`, URI/data, filename y descripción opcional;
- `message.metadata`;
- lista `agents` si existe.

### Synthetic

Se proyecta como role `user` con su metadata.

### System

Se proyecta con `Message.system(message.text)`.

### Shell

Se proyecta como role `user`:

```text
Shell command: <command>

<output>
```

manteniendo metadata.

### Assistant

La proyección distingue modelo/proveedor actual frente al que originó el mensaje.

**CONFIRMADO EN `dev`:** si provider e ID de modelo coinciden y el mensaje no contiene error, puede reutilizarse provider metadata de:

- reasoning;
- tool calls;
- tool results.

Si el modelo cambió:

- reasoning textual se degrada a `text` para preservar contenido semántico;
- metadata provider-specific que podría no ser portable deja de reutilizarse.

Esta es una pieza clave de continuidad cross-model.

### Tool calls y results

Se preservan:

- call ID;
- tool name;
- input parseado cuando es posible;
- si la tool fue ejecutada por el proveedor;
- provider metadata cuando es reutilizable.

Para tools provider-executed, call y result pueden permanecer dentro del assistant message. Para tools locales, los results se proyectan como mensajes tool separados.

### Compaction

Se proyecta como role `user` dentro de `<conversation-checkpoint>` y transporta `message.metadata`.

### Agent/model switched

**CONFIRMADO EN `dev`:** `agent-switched` y `model-switched` devuelven `[]`; son eventos de control de sesión, no contenido que deba ver el modelo directamente.

## Metadata de transporte / provider

Además del contenido del prompt, `LLM.request()` recibe metadata técnica no textual.

### HTTP headers

**CONFIRMADO EN `dev`:** el runner añade:

- `x-session-affinity: <session.id>`;
- `X-Session-Id: <session.id>`;
- `x-parent-session-id: <parentID>` cuando la sesión tiene padre.

Estos headers pertenecen al envelope de transporte y no son mensajes visibles para el modelo.

### Prompt cache key

El runner deriva un `promptCacheKey` de la sesión y lo pasa en:

```text
providerOptions.openai.promptCacheKey
```

Si el ID cumple el formato `ses_<64 hex>`, usa sólo el hash; de lo contrario usa el ID completo.

**CONFIRMADO EN `dev`:** existe por tanto una afinidad/cache identity por sesión para el provider OpenAI-compatible, separada del contenido semántico del prompt.

## Metadata de mensajes

`to-llm-message.ts` conserva `message.metadata` en user, synthetic, shell, assistant y compaction. También conserva metadata provider-specific en reasoning/tool parts únicamente cuando considera seguro reutilizarla con el mismo modelo.

**INFERENCIA, confianza alta:** esta división evita dos extremos problemáticos:

- perder continuidad técnica cuando el mismo proveedor puede reaprovechar opaque/provider state;
- enviar metadata de continuation incompatible cuando el modelo/proveedor cambió.

## Historia que realmente llega al modelo

`packages/core/src/session/history.ts` calcula la ventana durable del runner a partir de dos fronteras:

- `baselineSeq` del `SessionContextEpoch`;
- secuencia de la compaction más reciente.

### Sin compaction

Los system messages hasta `baselineSeq` se omiten porque ya están absorbidos en `system.baseline`. Se mantienen mensajes no-system y system updates posteriores.

### Con compaction

Se cargan:

- la compaction y todo lo posterior;
- además, system updates posteriores al `baselineSeq` cuando sea necesario.

**CONFIRMADO EN `dev`:** no se reenvía todo el historial antiguo. El checkpoint sustituye la porción compactada y el baseline sustituye el contexto system ya absorbido.

## Continuidad entre turns

La continuidad no depende de una sola técnica. Resulta de la combinación de:

1. persistencia de mensajes de sesión;
2. context epoch con baseline/snapshot;
3. deltas `system` para cambios de fuentes;
4. compaction checkpoint + recent tail;
5. provider metadata reutilizable cuando el modelo no cambia;
6. IDs estables de tools/messages;
7. re-resolución del modelo y rematerialización de tools en cada attempt.

## Cambio de agente

**CONFIRMADO EN `dev`:** cada nuevo turn vuelve a ejecutar `agents.select(session.agent)` y vuelve a cargar `skillGuidance.load(agent)`. Por ello un cambio durable de agente puede modificar:

- `agent.info.system`;
- permissions/tools materializadas;
- skill guidance;
- step limit.

El evento `agent-switched` no se añade como texto al modelo; sus efectos se observan mediante la nueva configuración del siguiente request.

## Cambio de modelo

Análogamente, `model-switched` no se convierte en mensaje textual. El runner resuelve el nuevo modelo y `toLLMMessages()` adapta el historial:

- mantiene texto;
- conserva reasoning como reasoning sólo si el modelo coincide;
- elimina metadata provider-specific no portable si cambia.

Esto reconstruye una continuidad semántica aunque cambie la representación protocolar.

## Steering y queued prompts

`SessionInput` permite promociones `steer` y `queue` antes de construir el request. Si se promueve input, el step counter puede reiniciarse.

La branch `queue-steer-prompts` es adyacente a este flujo: trata admission/UI de follow-ups. **No es una rama sobre system-prompt construction**, aunque influye en qué input queda durablemente disponible para un provider turn futuro.

## Evolución desde `SessionPrompt`

### `refactor/session-prompt-parts`

Commit representativo `1c8f84c4be6d4d0278c41bd725ea8411d32f1002` — `refactor(session): extract prompt parts`, seguido por simplificaciones de nombres.

**CONFIRMADO EN BRANCH:** marca el inicio de una descomposición del antiguo `SessionPrompt` en piezas con responsabilidades más pequeñas.

### `session-prompt-history`

**CONFIRMADO COMO LÍNEA EVOLUTIVA:** extrae/separa responsabilidades relacionadas con la selección/proyección de historia del prompt loop. En `dev` esa frontera es visible como `SessionHistory` + `toLLMMessages`.

### `feat/workspace-prompt-files`

**CONFIRMADO COMO LÍNEA HISTÓRICA:** introducía archivos de prompt/contexto a nivel workspace. La idea durable que llega a la arquitectura moderna no es necesariamente la API concreta de esa branch, sino que el workspace sea una fuente explícita de contexto y no un detalle global implícito.

### `layer-node-sprompt`

Pertenece a la línea de refactor Effect/layers del viejo SessionPrompt. Es relevante como evidencia de desacoplamiento, pero el análisis arquitectónico de esos layers corresponde principalmente al agente dedicado a Effect/refactors. Aquí sólo se usa para marcar la migración desde un monolito hacia services colaboradores.

## Invariante de autoridad

**INFERENCIA, confianza alta:** el pipeline actual impone tres dominios de autoridad distintos:

```text
SYSTEM AUTHORITY
  agent system + system-context baseline + system deltas

CONVERSATIONAL MEMORY
  user/assistant/tools/shell/synthetic

COMPRESSED HISTORY
  conversation-checkpoint como user
```

La frontera evita que un resumen o un output de tool pueda ascender accidentalmente al mismo nivel de autoridad que las instrucciones privilegiadas.

## Conclusión

El prompt final es mejor entendido como una **vista materializada de estado durable**. El runner reconstruye esa vista antes de cada provider turn usando fuentes, snapshots, historia proyectada, modelo y tool registry. La continuidad entre turns surge de persistir los componentes semánticos necesarios y volver a ensamblarlos, no de conservar una cadena de prompt completa de un turno al siguiente.