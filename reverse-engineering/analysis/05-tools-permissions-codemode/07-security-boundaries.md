# Seguridad y trust boundaries del tool runtime

## Tesis

El sistema de tools de OpenCode aplica defensa en profundidad. Ningún componente aislado —registry, frontend approval, schema o sandbox— constituye por sí solo toda la política de seguridad.

El pipeline reconstruido en `dev` es:

```text
Tool discovery
    |
    v
Tool visibility filtering
    |
    v
Provider/schema projection
    |
    v
LLM emits call
    |
    v
ToolPart lifecycle
    |
    v
Contextual permission check(s)
    |
    v
Side-effect boundary
    |
    +--> process
    +--> filesystem
    +--> MCP/capability
    +--> subagent session
    |
    v
bounded result / attachments
    |
    v
history/context reduction
```

## Boundary 1 — Tool publication

Antes de que el modelo pueda pedir una tool, `session/llm/request.ts` elimina tools globalmente disabled por el ruleset y por overrides del usuario.

### Propiedad

Una capability denegada globalmente puede no formar parte siquiera del schema enviado al modelo.

### Limitación

La visibilidad no puede sustituir permisos contextuales: un `shell` permitido en general necesita autorización diferente según comando/path, y `task` según subagent type.

## Boundary 2 — Input schema

`Tool.define()` valida parámetros y `tool/json-schema.ts` proyecta schemas compatibles con proveedores.

### Propiedad

Los inputs pasan por una representación tipada antes de llegar a implementaciones sensibles.

### Limitación

Un input válido por schema puede seguir siendo peligroso semánticamente. Por eso filesystem/shell realizan checks posteriores.

## Boundary 3 — Runtime approval

`Permission.Service.ask()` es la autoridad para decisions interactivas.

### Propiedades

- policy `deny` falla antes del side effect;
- `ask` suspende el Effect mediante `Deferred`;
- una respuesta humana desbloquea el backend, no sólo la UI;
- `always` está acotado a patterns ofrecidos;
- pendientes relacionados se reevalúan bajo el nuevo ruleset.

### Riesgo de configuración

Como `dev` usa last matching rule, ordenar incorrectamente rules puede ampliar/restringir permisos inesperadamente. El ordering debe tratarse como parte del security contract.

## Boundary 4 — Filesystem containment

`tool/external-directory.ts` verifica si el target pertenece a la instancia/worktree y pide `external_directory` si queda fuera.

### Consumidores importantes

- shell;
- edit;
- apply_patch;
- otras tools de lectura/escritura con targets externos.

### Propiedad

La autorización de la tool principal y la autorización para abandonar el workspace son dimensiones separadas.

## Boundary 5 — Edit authorization over computed intent

`edit` y `apply_patch` calculan el diff/targets antes de pedir `edit`.

### Propiedad

Se puede aprobar la consecuencia material del cambio, no únicamente el texto abstracto de la instrucción del modelo.

### `apply_patch`

El orden es particularmente robusto:

```text
parse -> resolve -> validate -> compute diff -> ask -> mutate
```

Esto reduce cambios parciales producidos antes de una denegación.

## Boundary 6 — Process execution

`shell.ts` cruza el límite más directo hacia el sistema operativo.

Controles observados:

- parse/scanner de command policy;
- patrones de permission;
- external-directory checks;
- cwd/environment administrados;
- AbortSignal;
- timeout;
- kill + forced kill;
- streaming/output cap.

### Propiedad

Una tool call cancelada no debería dejar indefinidamente un child process fuera del lifecycle del agente.

### Evidencia histórica

Las branches `shell-exit-*`, `shell-spawn-ready`, `fix/bash-tool-settlement`, `interrupt-tool-*` muestran que settlement/cancel fueron problemas explícitos de ingeniería.

## Boundary 7 — Code Mode capabilities

Code Mode no entrega un `eval` general con acceso irrestricto a Node/Bun. El host crea un catálogo explícito de child-tools.

Controles:

- prefiltrado con `Permission.visibleTools()`;
- child capability tree namespaceado;
- `ctx.ask()` en cada child call;
- schema MCP;
- timeout/progress;
- AbortSignal.

### Propiedad

El programa generado por el modelo sólo puede producir efectos a través de capabilities entregadas por el host.

### Interpretación

Es un sandbox de **capabilities**, no sólo un sandbox sintáctico.

## Boundary 8 — Subagent privilege propagation

`deriveSubagentSessionPermission()` impide interpretar spawning como copia completa de privilegios.

Se conservan:

- parent session denies;
- `external_directory`.

Las capabilities positivas proceden del agente hijo, y `task`/`todowrite` se cierran por defecto cuando no se declaran.

### Propiedad

Delegar trabajo a otro agente no borra restricciones negativas críticas de la sesión padre.

## Boundary 9 — Tool result size

Outputs enormes son también un riesgo operacional:

- consumo de memoria;
- context flooding;
- provider request overflow;
- degradación de replay.

`Truncate.Service` limita lo que se devuelve al contexto y salva el resultado completo temporalmente en disco. Shell hace spill incremental.

### Propiedad

Se preserva auditabilidad/acceso al output sin exigir que todo el contenido entre en el context window.

## Boundary 10 — Tool call repair

Calls mal formadas del modelo no se ejecutan ciegamente.

`session/llm.ts`:

- corrige nombres por lowercase únicamente cuando encuentra una tool real correspondiente;
- de lo contrario convierte la llamada a `invalid`.

### Propiedad

El repair de protocolo no inventa automáticamente una capability nueva a partir de un nombre desconocido.

## Boundary 11 — Lifecycle/persistence

ToolPart mantiene estado explícito y persistido.

Esto importa para seguridad/consistencia porque una interrupción o crash puede dejar evidencia de que una llamada llegó a `running` sin `completed`.

`message-v2.ts` distingue resultados interrumpidos para replay, evitando tratar indiscriminadamente un estado abandonado como una nueva acción pendiente.

## Authority map

| Decisión | Autoridad principal |
|---|---|
| qué tools existen | `ToolRegistry.Service` + MCP/plugins/config |
| qué tools ve el modelo | `LLMRequestPrep.resolveTools` + permissions |
| input estructuralmente válido | Tool schema / provider projection |
| si una acción contextual se permite | `Permission.Service` |
| si un path externo puede tocarse | `external_directory` policy |
| si el proceso sigue vivo | shell/process lifecycle |
| qué puede hacer Code Mode | child capability catalog + per-call permission |
| qué privilegios recibe subagent | child agent policy + selective inherited denies |
| cuánto output vuelve al contexto | truncation + message replay reduction |
| qué estado se persiste | Session/ToolPart processor |

## Qué NO debe considerarse autoridad

### UI de permisos

Modal/panel/keyboard shortcuts presentan una decisión. No son el enforcement point.

### Tool description

Una descripción guía al modelo, pero no reemplaza validación/policy.

### System prompt

Instruir al modelo a no ejecutar acciones peligrosas no sustituye `deny/ask` ni boundaries de filesystem/process.

### Tool visibility únicamente

Ocultar una tool al modelo reduce superficie, pero custom flows/workflow adapters todavía deben preservar authorization en el punto de ejecución.

## Posibles failure modes derivados del diseño

Estas son **inferencias**, no vulnerabilidades demostradas.

### 1. Ruleset ordering mistake

Dado last-match, una regla añadida al final puede sobreescribir otra más específica previa. Config generators deben preservar intención y tests de orden.

### 2. Parser mismatch en shell

Un scanner imperfecto puede no comprender un comando. La línea `shell-tool-policy` muestra consciencia explícita de comandos opacos y fallback de policy.

### 3. Path canonicalization/symlinks

La existencia de `patch-symlink-policy` demuestra que paths lógicos vs targets reales son una superficie a vigilar. Las resoluciones/normalizaciones actuales forman parte esencial del boundary.

### 4. New execution path bypassing hooks

Cada nuevo adapter que ejecute una tool directamente debe reproducir permission/hooks/abort. Code Mode y GitLab workflow model contienen código explícito para preservar estas invariantes.

### 5. Orphaned running state

Abort/crash entre side effect y persistencia final puede producir states incompletos; de ahí guards de replay y ramas históricas de settlement/interruption.

## Invariantes recomendadas para futuras auditorías

A partir del diseño reconstruido, cualquier nuevo runtime/tool debería satisfacer:

1. No producir side effects antes de autorización contextual aplicable.
2. Propagar cancelación hasta el recurso externo real.
3. No permitir que una capability exterior implique automáticamente todas las interiores.
4. Autorizar paths fuera de la instancia por separado.
5. Persistir estados terminales/error de forma distinguible.
6. Acotar output que vuelve al modelo.
7. Mantener schemas de provider como proyección del contrato interno, no como única validación.
8. Conservar restricciones negativas al delegar/spawnear.
9. Hacer visible en metadata suficiente información para review/audit sin depender del frontend.

## Conclusión

La seguridad del tool runtime de OpenCode está repartida deliberadamente entre catálogo, schema, policy, lifecycle y boundaries concretos de side effects. La arquitectura más importante que revelan las branches históricas es que OpenCode fue desplazando enforcement desde convenciones implícitas hacia servicios y adapters explícitos: `ToolRegistry`, `Permission.Service`, `external_directory`, process settlement y capability-scoped Code Mode.