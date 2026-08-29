# Sincronización entre backend y clientes

## 1. Superficies de sincronización

La sincronización de Session no depende de un único mecanismo. En `dev` convergen:

- queries de estado proyectado (`SessionStore`, Session APIs);
- event streams live/publicados;
- historial durable por Session;
- bridge de compatibilidad V2 → eventos legacy;
- SDK/protocolo generado.

Esto permite bootstrap + seguimiento incremental.

## 2. Bootstrap mediante proyecciones

Un cliente puede leer Sessions/messages materializados desde SQLite a través de APIs/SDK.

`session-store-reads`, commit `bc922b8ac590a48ae0fa4b14919c04f2886022d5`, formaliza el read side:

- Session list estable y paginado;
- messages paginados por `session_message.seq`.

**Inferencia:** el cliente no necesita replayar todo el event log desde cero para obtener una vista inicial utilizable.

## 3. Follow mediante events

Después del bootstrap, eventos publicados permiten mantener la UI/runtime actualizado.

La branch `session-event-stream` demuestra que existe un requisito explícito de ordering. Commit `5146f01e0a8826318056fdb38477fb0202c29099`: `fix(sdk): preserve session stream ordering`.

**Hecho confirmado:** los consumers no deben asumir que concurrencia de red/handlers conserva el orden lógico por sí sola.

## 4. Protocol event surface

`protocol-events`, especialmente `6ad5ce39beb88ababd7d429e45769ea61b2795d2`, define la “current event surface” y sus tests de group boundary.

Esto sitúa el catálogo de eventos en el contrato entre backend y consumidores, no sólo dentro del runner.

## 5. Published vs durable

`published-events`, commit `95f264e04eca61b4a81ef92ee29476556d299a68`, distingue durability de publication.

Consecuencia para clientes:

- algunos eventos live pueden ser útiles para animación/progreso pero no reaparecen en replay;
- boundaries durables sí pueden formar parte de historia/sync;
- un reconnect debe reconstruir el estado final desde durable/projections, no depender de haber recibido todos los deltas live.

Ejemplo principal: `Text.Delta` es live-only, `Text.Ended` contiene el texto replayable completo.

## 6. Bridge de compatibilidad

`packages/opencode/src/event-v2-bridge.ts` adapta eventos V2 a superficies legacy de Bus/eventos que todavía consumen componentes históricos.

**Hecho confirmado:** la migración V2 no exige que todos los clientes internos cambien simultáneamente.

**Inferencia:** el bridge funciona como anti-corruption layer durante la transición de protocolos.

## 7. SDK

Los cambios de event schemas regeneran tipos del SDK/OpenAPI. `normalize-step-event-versions` muestra de forma directa que cambiar la versión de `session.next.step.ended/failed` modifica tipos sync generados.

La branch `session-forking` también actualizó cliente generado al añadir `session.fork`.

**Hecho confirmado:** schema/protocolo es fuente para generar contratos cliente y, por tanto, cambios de eventos no quedan confinados al backend.

## 8. Ordering visible por el cliente

Debe distinguirse:

- orden de Session list: `time_updated` + tie-break ID;
- orden de timeline/messages V2: `seq`;
- orden de event history: aggregate `seq`;
- chunks live: orden de stream mientras la conexión permanece activa;
- diagnostics locales CLI: anchors/created time.

No existe un único “orden por ID” válido para todos esos dominios.

## 9. Reconnect y recovery

La combinación esperable es:

```text
reconnect
   |
   +--> cargar proyección / historial durable
   |
   +--> establecer stream live desde boundary conocido
   |
   +--> aplicar eventos posteriores en orden
```

**Inferencia:** `seq` ofrece el boundary natural para detectar qué historia durable precede a lo live, aunque cada API concreta puede encapsularlo de manera diferente.

## 10. Event batching

Batching puede reducir overhead de transporte, pero no puede alterar el orden lógico de cada Session. Los consumers deben poder desempaquetar/aplicar eventos respetando sequence.

**Inferencia:** batching pertenece al transport plane, mientras sequence pertenece al consistency plane.

## 11. Session model sync

Las branches `session-model-sync` y equivalentes forman parte de la familia que propaga cambios de selección/model metadata entre backend y consumidores. En el schema actual existen eventos durables como `ModelSwitched` y `AgentSwitched`, asociados a Session/message.

**Hecho confirmado:** model/agent selection es observable como cambio de Session history, no sólo variable local de UI.

## 12. Estado efímero del cliente

No todo lo visible debe sincronizarse desde backend. `session-prompt-drafts` mantiene drafts localmente por Session.

Esto marca una frontera práctica:

### Backend/durable o compartible

- Session metadata;
- messages/history;
- step/tool outcomes;
- agent/model switches;
- admitted prompts/moves;
- event sequence.

### Cliente efímero

- draft aún no enviado;
- transient presentation rows;
- scroll/layout/picker/tab state;
- ciertos streaming deltas una vez consolidado su `Ended` durable.

## 13. Consistencia y duplicate delivery

El event layer durable tiene semántica idempotente por ID/type/data/seq. Esto permite que capas de sync/replay reintenten sin aceptar silenciosamente una historia distinta.

**Inferencia:** la idempotencia durable reduce el impacto de duplicate delivery de transport, pero el cliente sigue necesitando handlers idempotentes o dedup cuando recibe superficies publicadas.

## 14. Arquitectura de sincronización reconstruida

```text
                 +----------------------+
                 | SQLite projections   |
                 | SessionStore         |
                 +----------+-----------+
                            |
                       bootstrap/read
                            |
                            v
+-------------+       +-----+------+       +----------------+
| Event log   |------>| backend /  |------>| SDK / clients  |
| aggregate   |       | protocol   |       | TUI/Desktop    |
| seq         |       +-----+------+       +----------------+
+------+------+             |
       |                     | published/live
       | replay/history      v
       +---------------> event stream

V2 bridge -> legacy Bus consumers durante migración
```

## 15. Conclusiones

### Confirmadas

- existen read models y event surfaces simultáneamente;
- Session stream tiene ordering explícito;
- protocolo define una superficie de eventos compartida;
- published no implica durable;
- SDK se regenera cuando cambia el contrato de eventos;
- existe bridge de compatibilidad V2/legacy;
- drafts pueden permanecer client-only.

### Inferencias

- la estrategia de sync es bootstrap por proyección + incremental events, no replay total obligatorio;
- durable full-value boundaries permiten recuperarse tras perder deltas live;
- `seq` es la pieza principal de consistency ordering, mientras transport, batching y UI pueden usar otras unidades.