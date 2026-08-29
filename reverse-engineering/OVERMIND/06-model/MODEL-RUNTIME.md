# Model Runtime — separar cognition propia de agent capability externa

## Preserve Overmind ModelService

Overmind ya posee:

```text
ModelService
 -> GenerationExecutor
 -> ModelTarget
 -> ModelBackend
```

Este boundary gobierna las inferencias de **Overmind**. Debe permanecer provider-neutral y conservar retry/recovery/continuation semantics actuales.

## OpenCode no es un ModelBackend

Un agente OpenCode delegado realiza múltiples inferencias, Tool calls, edits, shell commands y subagent work. Por tanto no representa una llamada de modelo normalizable como `ModelBackend.generate()`.

La relación correcta es:

```text
Overmind ModelService -> cognition del parent
Overmind ToolPort      -> OpenCode agent mission
OpenCode provider stack -> cognition interna del coding agent
```

No mezclar los dos runtimes de modelos.

## Provider ownership

OpenCode conserva su provider registry, auth, transforms, native/SDK runtime y model quirks. Overmind no obtiene valor copiándolos si solo consume el resultado de la misión.

Si Overmind necesita un backend adicional para **su propia cognition**, se implementa bajo `overmind/models/backends/<id>/` según el contract de Overmind.

## Target selection

Un eventual `agent.code` puede tener una configuración de OpenCode agent/model externa, pero no debe usar `ModelService` de Overmind para seleccionar cada model turn interno del coding agent.

Es posible permitir un profile bounded como `inspect|plan|implement`; el adapter/config traduce ese profile a la configuración OpenCode apropiada.

## Events

OpenCode normaliza sus propios model/runtime events. El adapter puede consumirlos para progreso, pero no necesita convertirlos uno-a-uno a `ModelEvent` Overmind.

Cuando Overmind formalice EVENTS, conviene surfaced solo semántica de delegación:

```text
delegation.progress
delegation.permission_waiting
delegation.completed
```

No `provider-token-delta-from-child-agent` salvo una UX específica que realmente lo necesite.

## Usage/cost

Mantener accounting separado:

- Overmind model usage pertenece a ModelService/GenerationExecutor.
- OpenCode mission usage, si se expone, pertenece a DelegationResult metadata/telemetry.

No agregar ambos como si fueran un único provider call.
