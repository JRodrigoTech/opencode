# State machine, Steps y lifecycle de ejecución

## 1. No existe una única state machine

El comportamiento de una Session se reparte en capas con responsabilidades distintas:

```text
SessionRunState
  └─ ownership del runner activo / cancelación / busy-idle

SessionProcessor (legacy path)
  └─ transforma stream LLM en Message Parts y resultados

Session runner V2
  └─ orquesta logical steps, retry, compaction y continuation

Durable Session Events
  └─ registra boundaries replayables del timeline
```

**Hecho confirmado:** estas capas tienen estados diferentes y no deben colapsarse en una sola enumeración.

---

## 2. SessionRunState: ownership de ejecución

Referencia: `packages/opencode/src/session/run-state.ts`.

El servicio mantiene:

```ts
Map<SessionID, Runner<WithParts>>
```

encapsulado en `InstanceState`.

Operaciones principales:

- `assertNotBusy(sessionID)`
- `cancel(sessionID)`
- `ensureRunning(sessionID, onInterrupt, work)`
- `startShell(...)`

### Transiciones observables

Al crear un `Runner`:

```text
onBusy -> SessionStatus.set(busy)
onIdle -> remove runner + SessionStatus.set(idle)
onInterrupt -> callback de settlement/recovery
```

`cancel()` también cancela background jobs asociados directa o transitivamente a la Session y luego cancela el runner.

**Hecho confirmado:** la exclusión de trabajo simultáneo por Session se implementa con presencia/estado del runner, no mediante una columna durable `busy`.

**Inferencia:** este diseño evita que un crash deje una Session permanentemente marcada como ocupada; tras reiniciar el proceso, el estado ejecutable se reconstruye desde ausencia de runner y desde historia durable.

---

## 3. SessionStatus: estado efímero publicado

Referencia: `packages/opencode/src/session/status.ts`.

`SessionStatus` mantiene otro `Map<SessionID, Info>` en memoria. `get()` devuelve `idle` si no hay entrada.

`set()`:

1. publica `SessionStatusEvent.Status` mediante `EventV2Bridge`;
2. si el nuevo estado es `idle`, publica además `Idle` y elimina la entrada;
3. en otros estados, actualiza el Map.

### Interpretación

```text
estado runtime       publicación live/bridge
-------------        -----------------------
busy          -----> status event
retry         -----> status event
idle          -----> status event + idle event + delete in-memory state
```

**Hecho confirmado:** status es observable por clientes, pero no equivale al event log durable de mensajes/steps.

---

## 4. SessionProcessor: reducción del stream LLM

Referencia: `packages/opencode/src/session/processor.ts`.

El processor crea un contexto mutable por intento con:

- `assistantMessage`
- `toolcalls`
- `shouldBreak`
- `snapshot`
- `blocked`
- `needsCompaction`
- `currentText`
- `reasoningMap`

El método `process()` consume eventos `LLMEvent` y actualiza Message/Parts.

### 4.1 Reasoning

Transición reconstruida:

```text
reasoning-start
  -> crea ReasoningPart(text="", time.start)

reasoning-delta
  -> concatena texto local
  -> publica PartDelta(field="text")

reasoning-end
  -> fija metadata final
  -> time.end
  -> updatePart completo
```

Los deltas huérfanos —sin `reasoning-start`— se descartan.

**Hecho confirmado:** durante streaming existe una combinación de estado local acumulado + deltas publicados, y al final se persiste la forma completa.

### 4.2 Tool calls

Estado inicial:

```text
tool-input-start/delta/end
       |
       v
ensureToolCall()
       |
       v
ToolPart(status=pending)
```

Cuando llega `tool-call`:

```text
pending -> running
           input = parsed input
           time.start = now
```

Terminales:

```text
running -> completed
           output, metadata, attachments, time.end

running -> error
           error, metadata parcial, time.end
```

`Deferred` permite señalar settlement de llamadas en vuelo.

También existe detección de doom loop: si los últimos tres Parts son la misma tool con el mismo input, se solicita permiso `doom_loop`.

### 4.3 Bloqueo y rechazo

Si un tool falla por `PermissionV1.RejectedError` o `Question.RejectedError`, el processor puede marcar `blocked` en función de `shouldBreak`.

**Inferencia:** el rechazo de interacción humana se trata como semántica de control del step, no simplemente como error técnico de tool.

---

## 5. Step como boundary lógico

En la línea V2, un Step no es sólo una llamada al provider. Agrupa:

- preparación de contexto;
- provider invocation;
- reasoning/text/tool activity;
- settlement de tools;
- usage/cost;
- snapshot;
- política de retry;
- decisión de compactar;
- continuation.

Los durable events actuales incluyen:

- `session.next.step.started`
- `session.next.step.ended`
- `session.next.step.failed`

Referencia: `packages/schema/src/session-event.ts`.

### Started

Contiene al menos:

- `sessionID`
- `assistantMessageID`
- `agent`
- `model`
- snapshot opcional
- timestamp

### Ended

Contiene:

- `assistantMessageID`
- `finish`
- `cost`
- tokens input/output/reasoning/cache
- snapshot/files opcionales
- timestamp

### Failed

Contiene:

- `assistantMessageID`
- error estructurado
- timestamp

**Hecho confirmado:** el settlement de Step es un boundary durable, distinto de los fragmentos de stream internos.

---

## 6. Evolución `step-settlement`

Commit: `fe64ef154caabd6d5e671184f4c69c94f1128958`.

La branch consolidó condiciones dispersas en una clasificación terminal única:

```text
Interrupted
ProviderFailed
ToolInfraFailed
Clean
```

### Semántica reconstruida

#### Interrupted

- el intento no puede considerarse completado normalmente;
- se cierran/fallan elementos que hayan quedado sin settlement;
- incluye interacciones abortadas/rechazadas según la causa.

#### ProviderFailed

- la generación falló a nivel provider/model;
- tool calls registrados pueden quedar huérfanos y necesitan normalización/failure antes de continuar o terminar.

#### ToolInfraFailed

- el provider pudo producir una llamada, pero la infraestructura de ejecución no pudo completar la semántica normal del tool.

#### Clean

- puede producirse un `Step.Ended` con usage/cost/final finish.

**Hecho confirmado:** la clasificación se realiza una sola vez y gobierna el settlement posterior.

**Inferencia:** esta refactor reduce un riesgo clásico en runtimes de agentes: que assistant message, tools y step terminen con estados terminales incompatibles entre sí.

---

## 7. Evolución `step-machine`

Commit: `7a0b4299e527b4f5d45bc3162212b4393907f08e`.

La branch extrae la orquestación a una state machine explícita.

### Comandos

- `Invoke`
- `Stop`
- `StopAndJoin`

### Runtime events

- input externo;
- `InvocationExited`;
- `InvocationsStopped`.

### Decisión del reducer

```text
Continue(nextState, commands)
Done(output)
```

### Generaciones

Cada invocación mantiene un `generation`. Si llega un `InvocationExited` de una generación antigua, se ignora.

```text
invoke generation 1
cancel/restart
invoke generation 2
late exit generation 1 -> stale -> ignored
```

**Hecho confirmado:** esta técnica evita que un completion tardío de un fiber cancelado corrompa el state actual.

### StopAndJoin

La máquina invalida exits individuales, interrumpe/espera las invocaciones y después emite un evento agregado de stopped/joined.

**Inferencia:** la atomicidad lógica de “cancelar todo y decidir después” es necesaria para compaction, interruption y cambios de fase donde respuestas tardías del provider/tool no deben reabrir un step ya cerrado.

---

## 8. Retry y compaction como decisiones de orchestration

La presencia de `Result = "compact" | "stop" | "continue"` en el processor legacy y la migración posterior a `step-machine` muestran una misma estructura conceptual:

```text
attempt settles
    |
    +--> continue -> próximo step/continuation
    +--> compact  -> checkpoint/compaction -> nuevo contexto
    +--> retry    -> nueva invocación controlada
    +--> stop     -> terminal
```

**Hecho confirmado:** overflow/compaction y retry no son simples excepciones lanzadas fuera del runtime; forman parte del control de continuidad de la Session.

---

## 9. Recovery de tool calls incompletos

`message-v2.ts` transforma antiguos Parts `pending` o `running` en un tool result de error al reconstruir model messages:

```text
[Tool execution was interrupted]
```

Esto evita enviar a providers un `tool_use` sin respuesta correspondiente.

**Hecho confirmado:** una Session puede reanudarse después de una interrupción aun si el proceso murió entre tool call y tool result.

**Inferencia:** la state machine persistida no exige que todas las transiciones intermedias hayan terminado físicamente; dispone de reglas de normalización para convertir estados no terminales históricos en un transcript provider-valid.

---

## 10. Máquina conceptual consolidada

```text
IDLE
  |
  | prompt/resume
  v
BUSY / RUNNER OWNED
  |
  v
STEP STARTED
  |
  +--> reasoning/text deltas
  +--> tool pending -> running -> completed/error
  |
  v
ATTEMPT SETTLEMENT
  |
  +--> interrupted ------> STOP / normalize incomplete state
  +--> provider failed ---> RETRY or STOP
  +--> tool infra fail ---> RETRY or STOP
  +--> clean ------------> STEP ENDED
                              |
                              +--> continuation -> next STEP
                              +--> compaction --> compacted context -> next STEP
                              +--> done --------> IDLE
```

Esta figura es una **síntesis arquitectónica**, no una enum literal de un único archivo.