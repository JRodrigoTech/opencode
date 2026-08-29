# Inventario y agrupación de branches

## Criterio de agrupación

Las branches se agrupan por **invariante arquitectónico afectado**, no sólo por prefijo del nombre. Una misma branch puede tocar UI y backend; se considera de núcleo únicamente si modifica contratos de Session, Message, Event, persistence, replay o ejecución.

## Familia A — Session lifecycle, history y control

Branches principales:

- `session-forking`
- `session-history-api`
- `session-store-reads`
- `session-version-sentinel`
- `session-model-sync`
- `session-prompt-history`
- `session-prompt-drafts`
- `explicit-session-selection`
- `explicit-turn-api`
- `archive-session`
- `session-archival`
- `session-list-index`
- `orphan-session-recovery`

### Propósito reconstruido

Esta familia explora la separación entre:

- identidad y metadata de Session;
- historia de mensajes;
- parent/child y fork;
- lectura proyectada;
- selección explícita de Session/turn;
- archivo/listado;
- recuperación de Sessions cuyo location/worktree ya no está disponible.

### Evidencia relevante

`session-store-reads` contiene el commit `bc922b8ac590a48ae0fa4b14919c04f2886022d5`, `refactor(core): move projected Session reads into Store`. El cambio traslada `list()` y `messages()` desde el servicio Session hacia `SessionStore`, incluyendo paginación por anchor y por `seq` de `session_message`.

**Hecho confirmado:** en esa evolución se reconoce explícitamente un boundary read-side para datos proyectados.

**Inferencia:** `Session.Service` se orienta progresivamente hacia policy/commands/lifecycle, mientras `SessionStore` asume queries y representación materializada.

`session-version-sentinel`, commit `95d5a4a56266fe3c9738c9720f9b117429a8d7ce`, desacopla la versión de Session de la versión de instalación y usa temporalmente `"unknown"`.

**Inferencia:** el campo `version` se estaba convirtiendo en metadata de compatibilidad/historia y no debía depender implícitamente del proceso host actual.

`session-prompt-drafts`, commit `ea74a84ea37df6ee75789e27ebda1ec5b25ab7f3`, reemplaza un único `stashed` global por `Map<sessionID, draft>` en TUI.

**Hecho confirmado:** los drafts inspeccionados son efímeros y client-side; no forman parte del modelo durable de Session.

## Familia B — Message model, identity y row codecs

Branches principales:

- `message-v3`
- `message-row-codec`
- `refactor/core-v2-message-identity`
- ramas `message-*` relacionadas con updates, identity y schema

### Hitos

#### `refactor/core-v2-message-identity`

Commit principal inspeccionado: `f2d133edf7430f3229a84c29d6b0e1690f1f2a60`, `refactor(core): separate v2 message identity`.

Cambios confirmados:

- `Event.ID` pasa a exigir prefijo `evt_`.
- aparece `SessionMessageID.ID` con prefijo `msg_`.
- se añaden `fromEvent()` y `toEvent()` para mapping determinista.
- projectors y inbox dejan de usar Event ID directamente como Message ID.
- una migración beta resetea `session_input`, `session_message`, `event` y `event_sequence`, preservando `session`, `message` y `part` V1.

**Interpretación:** esta branch marca una ruptura conceptual importante: Event y Message ya no son dos vistas del mismo identificador, aunque exista una relación determinista entre ellos.

#### `message-row-codec`

Commit principal: `ffaaad89a44fd4069c8f5f232c9cfaea1163dc6b`, `refactor(core): centralize session message rows`.

Introduce `SessionMessageRow` con:

- `decode()`
- `decodeSync()`
- `encode()`

Se reutiliza desde history, pending/inbox, projector, revert, store e importación de Sessions.

**Hecho confirmado:** la representación SQLite de Message queda encapsulada en un codec único.

**Inferencia:** se intenta evitar que schema de dominio y layout físico diverjan silenciosamente cuando se añaden o cambian tipos de mensajes.

## Familia C — Eventos publicados, protocolos y streams

Branches principales:

- `protocol-events`
- `published-events`
- `session-event-stream`
- `event-batching`
- `normalize-step-event-versions`
- ramas `event-*` y `published-*` relacionadas con transport/versioning

### Propósito reconstruido

- estabilizar un catálogo de eventos públicos/versionados;
- separar live events de durable events;
- ofrecer historial + seguimiento incremental;
- batch/replay/sync con orden por aggregate;
- mantener compatibilidad entre runtime interno y SDK/protocolo externo.

`normalize-step-event-versions`, commit `179a8748cb9ba02cf6a633ed28da8c4fd0b4e9c0`, demuestra que la numeración de versiones de eventos V2 no fue lineal: después de descartar historia experimental se rebasaron `session.next.step.ended` y `.failed` a `.1`. En `dev` esos settlements vuelven a declararse con versión durable 2.

**Conclusión:** una versión menor/mayor del event type sólo puede interpretarse dentro del contexto de compatibilidad y resets de la época.

## Familia D — Step state machine y settlement

Branches principales:

- `step-machine`
- `step-settlement`
- `normalize-step-event-versions`
- ramas `step-*` relacionadas con continuation, retry y settlement

### `step-settlement`

Commit principal: `fe64ef154caabd6d5e671184f4c69c94f1128958`, `refactor(core): classify step settlement once`.

Clasifica el terminal state del intento como:

- `Interrupted`
- `ProviderFailed`
- `ToolInfraFailed`
- `Clean`

Antes, múltiples booleanos y ramas de control decidían por separado qué hacer con tools, assistant y Step.Ended.

**Hecho confirmado:** esta branch reduce estados terminales implícitos a una clasificación explícita.

### `step-machine`

Commit principal: `7a0b4299e527b4f5d45bc3162212b4393907f08e`, `refactor(core): unify logical step orchestration in a state machine`.

Introduce una máquina genérica con:

- `Invoke`
- `Stop`
- `StopAndJoin`
- generaciones de invocación para ignorar exits stale;
- queue interna de runtime events;
- manejo explícito de interrupciones;
- separación `Decision = Continue | Done`.

El runner usa después `SessionStepMachine` para gobernar retry, compaction, continuation y assistantMessageID.

**Inferencia fuerte:** la arquitectura deseada separa dos niveles:

1. attempt/provider/tool execution;
2. logical Step orchestration.

## Familia E — Storage, SQLite y migraciones

Branches principales:

- `storage-v2-service`
- `sqlite`
- `message-row-codec`
- branches de migrations/event sequence/projection

### `storage-v2-service`

Commits observados incluyen:

- `b3d6f931484137d271039272a9cc756c741f9095` — `refactor(v2): share effect sqlite database layer`
- `71423b9a5825eed2a018b5cc118dcf26f19ed6f1` — `refactor(v2): keep test database setup explicit`
- `178840489f3c99b1a47f1aa94ca25585c8dd1e03` — documentación del bootstrap DB

**Hecho confirmado:** esta línea migra V2 hacia un servicio SQLite compartido y testeable.

### `sqlite`

Es una branch mucho más antigua y con commits de sincronización amplios. Se conserva como antecedente, pero no se usa como evidencia fina de la arquitectura actual salvo coincidencia con código/commits posteriores.

## Familia F — Replay, recovery y consistency

Branches principales:

- `replay-time-order`
- `orphan-session-recovery`
- branches de resume/recovery/session move

### `replay-time-order`

Commit `642d6c52c4ec43192c75c1adbd9df18554087218`.

Cambio confirmado:

- las filas locales del CLI guardan `created: Date.now()`;
- se deja de inferir cronología sólo por comparación lexicográfica de `messageID`;
- se usan anchors y timestamp para insertar diagnostics/local commits respecto a filas persistidas.

**Importante:** esto es replay de presentación/runtime local, no el replay durable de `EventV2`.

### `orphan-session-recovery`

Commits inspeccionados:

- `a3d3d6b67faf53880c3da83382a74f704d1b92ce` — recuperación tras borrado de child worktree;
- `44204f2057d76db34d5b855d598fbbed353ba7a6` — simplificación de orphaned blocker recovery;
- `0c371bdb7a03a67f642ed55170dc6fa111c3643d` — preservación de canonical locations y queued moves;
- `2cd7dd049d3119e4f54b572a10b68c8bdd332b8f` — evita recovery work innecesario.

**Hecho confirmado:** la Session puede sobrevivir a que la location física desaparezca; existen capas de recovery en host/server para mantener accesibles datos y blockers asociados.

## Familia G — State freshness y domains

Branches observadas:

- `state-read-freshness`
- `state-domains`

`state-read-freshness` está dominada por registries/config/plugin/skill freshness y no por Session durable. Se clasifica como **transversal/periférica** para este agente.

`state-domains`, commit `7374576ee61e61c76ad4e075dbca6eaba1297656`, desacopla state domains de config. Es relevante como boundary general, pero no modifica directamente el event-sourced Session model estudiado aquí.

## Branches UI-only o predominantemente de presentación

Ejemplos detectados en la familia amplia de `session-*`:

- tabs/session tabs
- picker/session picker
- scroll/timeline state
- layout/session panel
- visual archive controls

Estas branches se excluyen del núcleo salvo que cambien:

- API de Session;
- schema de Message/Event;
- persistence;
- replay;
- runtime state machine.

## Relación evolutiva resumida

```text
storage-v2-service
        |
        v
EventV2 + SQLite projections
        |
        +--> core-v2-message-identity
        |        |
        |        +--> msg_* / evt_* separados
        |
        +--> published/protocol events
        |        |
        |        +--> session-event-stream / sync
        |
        +--> step-settlement
        |        |
        |        +--> step-machine
        |
        +--> message-row-codec
        |
        +--> session-store-reads
        |
        +--> recovery / replay fixes
        v
       dev
```

No todas las flechas implican ancestry Git exacto; representan **dependencia conceptual/evolutiva** reconstruida a partir de commits y código actual.