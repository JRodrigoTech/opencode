# Sistema de permisos y aprobación

## Modelo vigente en `dev`

La implementación autoritativa vive principalmente en `packages/opencode/src/permission/index.ts`, con tipos de error legacy/compatibilidad en `packages/core/src/v1/permission.ts` y reglas propagadas desde agentes/sesiones.

## Ruleset

Una regla contiene tres campos conceptuales:

```text
permission + pattern -> action
```

Las acciones son:

- `allow`
- `deny`
- `ask`

### Precedencia confirmada

En `dev`, la evaluación usa **orden de reglas**. La última regla coincidente es la que decide el resultado.

Esto tiene consecuencias importantes:

- `merge()` no intenta resolver conflictos; concatena rulesets.
- El orden en el que agent/session/config añaden reglas forma parte de la semántica.
- Una regla específica sólo gana a una general si aparece después; no existe un ranking implícito por longitud o número de wildcards en el baseline actual.

### Experimento de specificity

La branch `nxl/fix-permission-specificity` contiene una alternativa explícita en `packages/core/src/permission.ts`.

**Confirmado en esa branch:** `select()` puntúa reglas por:

1. menor número de wildcards en `permission`;
2. mayor cantidad literal en `permission`;
3. menor número de wildcards en `pattern`;
4. mayor cantidad literal en `pattern`;
5. posición como desempate.

El diff aislado de esa branch modifica precisamente el algoritmo y tests/docs. Ese algoritmo **no es el de `dev`**, por lo que debe interpretarse como un intento de corregir ambigüedades de specificity que no terminó siendo la semántica vigente.

## `Permission.Service.ask()`

`ask()` es el punto central de autorización interactiva.

### Flujo reconstruido

```text
Tool.execute
   |
   v
ctx.ask(request)
   |
   v
evaluar todos los patterns
   |
   +-- deny --> PermissionDeniedError
   |
   +-- todos allow --> continuar inmediatamente
   |
   `-- queda ask
          |
          v
     registrar solicitud pendiente
          |
          v
     publicar Permission.Event.Asked
          |
          v
     esperar Deferred
          |
      +---+---+
      |       |
    reply   cancel/session teardown
```

**Confirmado:** el runtime no continúa la side effect mientras la solicitud queda pendiente. La espera se implementa con primitives de Effect (`Deferred`), por lo que la UI sólo presenta/contesta una decisión cuyo bloqueo real ocurre en backend/runtime.

## Respuestas humanas

Las respuestas distinguen, conceptualmente, tres clases de desenlace:

- permitir sólo esta invocación;
- rechazarla;
- permitir permanentemente los patrones ofrecidos por `request.always` dentro del alcance correspondiente.

### Semántica de `always`

`always` no es un bypass global. La propia tool decide qué patrones propone como reutilizables.

Al aceptar permanentemente una solicitud:

1. el runtime añade reglas `allow` para los patrones de `request.always`;
2. reevalúa solicitudes pendientes de la misma sesión;
3. sólo resuelve automáticamente las que, bajo el ruleset actualizado, resulten enteramente autorizadas.

Esta reevaluación es relevante para invocaciones paralelas: una aprobación puede desbloquear otra solicitud equivalente sin convertir la sesión en unrestricted.

## Rechazo vs policy deny

`packages/core/src/v1/permission.ts` modela errores diferenciados:

- `RejectedError`: el usuario rechazó esa tool call.
- `CorrectedError`: rechazo con feedback/corrección textual.
- `DeniedError`: una regla del ruleset impide la invocación.
- `NotFoundError`: no existe la solicitud de permiso indicada.

**Confirmado:** esto permite al agent loop distinguir una política preexistente de una decisión interactiva humana. No deben documentarse ambas como el mismo “permission denied”.

## Filtrado de tools antes del modelo

`packages/opencode/src/session/llm/request.ts` llama a `Permission.disabled()` antes de proyectar las tools al LLM.

Esto constituye una primera capa:

```text
ruleset -> tools visibles al modelo
```

Pero no autoriza la invocación concreta. El segundo nivel ocurre dentro de la tool:

```text
tool visible -> llamada concreta -> ctx.ask(permission, patterns)
```

Ejemplos:

- `task`: patrón = tipo de subagente.
- `edit` / `apply_patch`: patrones = paths relativos afectados.
- `shell`: patrones derivados del comando y directorios alcanzados.
- Code Mode: patrón por child-tool MCP.
- `external_directory`: patrón del directorio fuera del worktree/instance boundary.

## Herencia y composición

### Agent + session

Los callers fusionan permisos de agente y sesión mediante `Permission.merge(...)`. Dado que la semántica es ordered-last-match, la posición de cada conjunto es significativa.

### Subagentes

`packages/opencode/src/agent/subagent-permissions.ts::deriveSubagentSessionPermission()` documenta de forma explícita un boundary que sería fácil interpretar mal.

**Confirmado:** al crear la sesión hija se heredan del padre:

- reglas con `action === "deny"`;
- reglas de `external_directory`.

No se heredan ciegamente todos los `allow`/`ask` del agente padre. Las capacidades del subagente las determina su propio `Agent.Info.permission`.

Además, si el subagente no declara reglas para determinadas capacidades:

- `task` se deniega por defecto;
- `todowrite` se deniega por defecto.

`packages/opencode/src/tool/task.ts` aplica esa función al crear la child session y añade los deny defaults evitando duplicados.

### Interpretación

El sistema diferencia dos categorías:

1. **restricciones de contención de la sesión padre**, que sí atraviesan el boundary;
2. **capacidades funcionales del agente**, que se vuelven a determinar según el subagente.

Esto evita que seleccionar un subagente sea equivalente a clonar exactamente los privilegios del caller.

## External directory como permiso transversal

`packages/opencode/src/tool/external-directory.ts` centraliza un boundary de filesystem.

`assertExternalDirectoryEffect()`:

- resuelve/normaliza el path;
- comprueba si está contenido en la instancia actual;
- si está fuera, pide `external_directory` sobre un glob del directorio;
- adjunta metadata de filepath y parent directory.

Este permiso aparece transversalmente en shell, edit, apply-patch y otras tools con acceso a paths.

## Branches de evolución

### `permission-rework`

**Hecho:** es una branch histórica pequeña respecto a su época y contiene la línea `permission/next` que introduce/estructura el modelo de rulesets, requests pendientes y respuestas que posteriormente aparece refinado en `dev`.

**Inferencia:** representa la transición principal desde una autorización más ad-hoc hacia el service actual basado en `allow|deny|ask`.

### `nxl/fix-permission-specificity`

**Hecho:** implementa selección por specificity y actualiza tests/documentación. No sobrevivió como algoritmo vigente.

### `agent-permission-order`, `adjust-perm-array-logic`, `update-perms`

**Inferencia respaldada por nomenclatura y evolución:** pertenecen a la fase de estabilización de composición/orden de arrays de reglas. La presencia actual de semántica last-match explica por qué ordering se convirtió en parte crítica del contrato.

### `subagent-permission-inheritance`

**Hecho vigente relacionado:** `dev` contiene `agent/subagent-permissions.ts` con política explícita de herencia selectiva. Debido a que la topic branch acumuló una divergencia muy amplia, la evidencia más fiable es el código final superviviente en `dev`, no atribuir cada línea del diff agregado a esa branch.

### `auto-accept-permissions` / `feat/auto-accept-permissions`

**Inferencia:** exploran bypass/autoaccept en superficies controladas. Deben distinguirse del mecanismo `always`: el baseline vigente sigue conservando autorización backend y rulesets en vez de convertir approvals en una mera preferencia visual.

### UI: `permission-highlight`, `permission-modal-keyboard`, `permission-panel`, `kit/permission-flow-ux`

Estas branches se catalogan como experiencia de aprobación salvo evidencia contraria. No se usan como prueba de cambios en la autoridad de seguridad.

## Propiedades de seguridad confirmadas

- La UI no concede permisos por modificar estado local; debe responder al backend.
- `deny` puede impedir que una tool se publique al modelo y también abortar una petición contextual.
- Una aprobación persistente está acotada a patrones propuestos por la propia solicitud.
- Directorios externos forman una dimensión de autorización independiente de la tool principal.
- Los subagentes preservan restricciones parentales críticas en lugar de recibir automáticamente todos los allows del padre.

## Inferencias arquitectónicas

- La semántica last-match convierte el ruleset en una pequeña policy language ordenada, no en un set matemático de ACLs.
- La separación entre visibility filtering y runtime `ask` reduce tanto llamadas imposibles del modelo como escaladas debidas a inputs contextuales.
- El uso de `Deferred` revela que approval pertenece al scheduler/lifecycle de ejecución y no sólo al protocolo de UI.