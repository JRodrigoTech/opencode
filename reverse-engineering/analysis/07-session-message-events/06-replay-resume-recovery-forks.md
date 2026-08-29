# Replay, resume, recovery, forks y drafts

## 1. Cuatro conceptos que no deben mezclarse

En OpenCode aparecen operaciones similares desde la UI pero arquitectónicamente distintas:

- **replay**: reconstruir/reemitir historia ya producida;
- **resume**: reactivar ejecución sobre una Session existente;
- **fork**: crear una nueva Session clonando un prefijo de historia;
- **recovery**: restaurar operabilidad cuando recursos/runtime quedaron en un estado incompleto o desaparecieron.

## 2. Replay durable de EventV2

`packages/core/src/event.ts` soporta commit/import de eventos con sequence conocido.

Cuando `seq <= latest`, la operación no acepta silenciosamente cualquier payload: busca el evento persistido y comprueba que coincidan ID, type y data.

Si no coinciden se considera replay divergence.

Cuando el sequence esperado no encaja, también se rechaza el gap.

### Invariante

```text
(session A, seq 42) -> exactamente un hecho durable
```

Replay del mismo hecho es idempotente; otro hecho con el mismo sequence es inconsistencia.

## 3. Replay local del CLI

`replay-time-order`, commit `642d6c52c4ec43192c75c1adbd9df18554087218`, resuelve otro problema.

El CLI mezcla:

- rows persistidas del transcript;
- diagnostics/commits locales todavía no persistidos.

Antes intentaba situar ciertos rows comparando `messageID`. La branch añade `created: Date.now()` y usa anchors/timestamps.

El test cambia explícitamente a: no inferir chronology from message IDs.

**Hecho confirmado:** este replay es de presentación/runtime local; no es el algoritmo de consistencia durable de EventV2.

## 4. Resume

Resume mantiene la identidad de Session y vuelve a poner su runner en condiciones de procesar trabajo pendiente/admisible.

En el core moderno esto interactúa con:

- inbox/admission;
- runner/execution;
- session state;
- location;
- pending moves/prompts.

Resume no implica copiar la historia ni regenerar IDs.

### Diferencia con retry

Retry pertenece a un intento/Step. Resume es una operación sobre el lifecycle de la Session ya existente.

## 5. Fork

La implementación histórica `packages/opencode/src/session/session.ts` clona messages y parts con nuevos IDs y puede detenerse en `messageID`.

`session-forking`, commit `402e5999da6677e38b4d782911a7def821453be2`, implementa la variante V2 y expone `session.fork` al cliente/SDK.

### Propiedad esencial

La nueva Session obtiene su propia identidad. El fork no es replay in-place.

### Parent/child vs fork

- child: conserva `parentID` como relación estructural;
- fork: conserva historia derivada, no identidad/ownership padre-hijo como semántica necesaria.

## 6. Revert

Las rutas `session.revert.stage`, `.clear` y `.commit`, visibles también en la evolución del cliente generada alrededor de `session-forking`, representan otra forma distinta de manipular historia/working tree.

Un revert staged debe cerrarse/commit antes de admitir nuevo trabajo cuando la política así lo requiere; por ejemplo el commit histórico `42a3cf964597c3b4aa9226d4a5f0b2465c769cf2` corrige ese boundary.

**Inferencia:** revert es una transacción sobre estado de sesión/workspace, no una simple edición cosmética del transcript.

## 7. Orphan session recovery

`orphan-session-recovery` contiene una secuencia explícita de fixes/refactors:

- `a3d3d6b67faf53880c3da83382a74f704d1b92ce` — `recover cleanly after child worktree removal`;
- `44204f2057d76db34d5b855d598fbbed353ba7a6` — simplifica recovery de blockers huérfanos;
- `0c371bdb7a03a67f642ed55170dc6fa111c3643d` — preserva canonical locations y queued moves;
- `3ce067230647371b1dafe256f8ead03840157d39` — acota recovery de missing-location en server;
- `2cd7dd049d3119e4f54b572a10b68c8bdd332b8f` — evita trabajo de recovery innecesario.

### Qué revela

Una Session persistida puede sobrevivir a la desaparición de su location física. El server necesita seguir pudiendo acceder a blockers/forms/permissions e información suficiente para recuperar/mover la Session.

**Hecho confirmado:** availability del filesystem y durability de Session son concerns independientes.

## 8. Queued moves

Los commits de recovery muestran que los moves pueden entrar en el inbox y entregarse en un boundary seguro de ejecución.

Un move queued no debe perderse simplemente porque el destino coincide temporalmente con la location actual o porque existe una secuencia “away and back”.

**Inferencia:** el inbox representa intención ordenada; deduplicar sólo por estado actual puede destruir semántica temporal.

## 9. Session history API

`session-history-api` es una branch long-lived cuyo diff global contra `dev` resulta ruidoso. Se trata como línea evolutiva de exposición del historial, no como patch aislado.

El diseño que sobrevivió se reconoce mejor en:

- `SessionStore.messages()`;
- `session_message.seq`;
- event history/stream;
- SDK Session/event APIs.

**Inferencia:** la historia se separa progresivamente de “leer un array de messages” y se convierte en superficie paginada/streamable con ordering explícito.

## 10. Drafts

`session-prompt-drafts`, commit `ea74a84ea37df6ee75789e27ebda1ec5b25ab7f3`, mantiene drafts por Session en un `Map` de TUI.

### Consecuencia

- cambiar de Session puede stashear/restaurar el draft correspondiente;
- enviar/limpiar elimina esa entrada;
- no se observó persistencia durable backend de ese draft en la branch inspeccionada.

**Hecho confirmado:** el draft de esta línea es estado efímero de cliente.

No debe confundirse con:

- prompt admitted en inbox;
- message persistido;
- pending Session input.

## 11. Recovery de herramientas/steps

El settlement de Step también contiene recovery a nivel runtime: provider failure, interrupted tools y tool infrastructure failure se resuelven de forma diferente.

Esto complementa el recovery de Session/location:

```text
recovery externo: filesystem/location/server
recovery interno: provider/tool/step state
```

## 12. Conclusiones

### Hechos confirmados

- EventV2 replay valida identidad/tipo/data y sequence.
- el CLI tiene un replay local independiente basado en anchors/timestamps.
- resume conserva Session ID.
- fork crea otra Session y clona/remapea historia.
- orphan recovery permite operar con Sessions cuya location desapareció.
- drafts inspeccionados son client-side.

### Inferencias

- OpenCode evita usar IDs como reloj lógico; sequence y timestamps/anchors cumplen roles separados.
- recovery se diseña por boundaries: Session/location, inbox, Step y tools.
- el inbox durable/ordenado permite recuperar intención pendiente sin inferirla desde el estado materializado actual.