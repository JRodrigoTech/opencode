# AGENT 05 — Tools, permisos y Code Mode

## Alcance

Este análisis reconstruye el runtime de herramientas y el sistema de autorización de OpenCode tomando `dev` como baseline vigente y contrastándolo con las familias históricas de branches relacionadas con tools, permisos, shell/bash, patch/edit, schemas, outputs, approvals y Code Mode.

El análisis se ha realizado sobre el árbol de documentación `reverse-engineering`; no se modifica código de producto.

## Documentos

- [`01-tool-runtime.md`](./01-tool-runtime.md): registry, schemas, invocación, lifecycle y resultados.
- [`02-permissions.md`](./02-permissions.md): resolución de permisos, precedencia, aprobación/rechazo e herencia.
- [`03-code-mode.md`](./03-code-mode.md): runtime `execute`, catálogo MCP y confinamiento.
- [`04-shell-bash.md`](./04-shell-bash.md): shell/bash, análisis de comandos, streaming, cancelación y políticas.
- [`05-patch-edit.md`](./05-patch-edit.md): `apply_patch`, `edit`, autorización de mutaciones y límites de filesystem.
- [`06-branch-evolution.md`](./06-branch-evolution.md): inventario y agrupación de branches relacionadas, con su relación evolutiva frente a `dev`.
- [`07-security-boundaries.md`](./07-security-boundaries.md): trust boundaries y propiedades de seguridad reconstruidas.

## Resumen ejecutivo

### Arquitectura vigente confirmada

El runtime actual no es un único dispatcher. Se divide en varias capas:

1. `packages/opencode/src/tool/tool.ts` define el contrato común `Tool.Def`, `Tool.Context` y `Tool.ExecuteResult`, valida parámetros mediante Effect Schema, añade tracing y aplica truncado automático.
2. `packages/opencode/src/tool/registry.ts` materializa las tools builtin, custom y de plugins; decide su visibilidad según modelo, flags, MCP disponible y permisos.
3. `packages/opencode/src/session/llm/request.ts` proyecta las tools habilitadas hacia el proveedor/AI SDK y elimina las globalmente denegadas.
4. `packages/opencode/src/session/processor.ts` mantiene el estado persistido de cada invocación: `pending -> running -> completed|error`.
5. `packages/opencode/src/permission/index.ts` resuelve `allow|deny|ask`, crea solicitudes pendientes y espera una respuesta mediante `Deferred`.
6. Cada tool sensible vuelve a pedir autorización con patrones concretos antes de producir side effects. Ejemplos: `shell`, `edit`, `apply_patch`, `task` y cada child-tool MCP de Code Mode.
7. `packages/opencode/src/tool/truncate.ts` limita output persistido en el contexto y conserva el resultado completo temporalmente en disco cuando excede los límites.

### Hallazgos principales

- **La precedencia de permisos vigente es por orden, no por especificidad**: la última regla que coincide con `permission` y `pattern` gana. La branch `nxl/fix-permission-specificity` experimentó con un ranking por specificity, pero ese algoritmo no está en `dev`.
- **`ask` es una suspensión real del runtime**, no un estado de UI: `Permission.Service.ask()` crea una solicitud, publica el evento y bloquea el Effect sobre un `Deferred` hasta `reply()`.
- **`always` no significa “permitir todo”**. Añade reglas `allow` para los patrones de `request.always`; después reevalúa las demás solicitudes pendientes de la misma sesión y desbloquea únicamente las que queden totalmente autorizadas.
- **El rechazo es distinto del deny de policy**. `RejectedError`, `CorrectedError` y `DeniedError` modelan respectivamente rechazo humano, rechazo con feedback y denegación derivada del ruleset.
- **Los subagentes no heredan ciegamente todo el ruleset del padre**. `deriveSubagentSessionPermission()` conserva los `deny` de la sesión padre y `external_directory`, pero las capacidades del subagente proceden de su propio agente. `task` y `todowrite` quedan denegadas por defecto si el subagente no las declara.
- **Code Mode está confinado por construcción**: la tool exterior es `execute`; el intérprete recibe únicamente un árbol de child-tools MCP ya filtrado por permisos. Cada child call vuelve a pasar por `ctx.ask()` y por timeout/abort del cliente MCP.
- **Shell combina autorización semántica y boundary de filesystem**. Parsea el comando, deriva patrones autorizables y solicita por separado acceso a directorios externos. La ejecución se puede cancelar y tiene kill escalonado.
- **`apply_patch` sigue un patrón verify -> authorize -> mutate**: parsea todos los hunks, calcula targets/diff, valida directorios externos, pide permiso `edit` y sólo entonces escribe/elimina/mueve archivos.
- **El output tiene dos mecanismos diferentes de reducción**: truncado operativo en `tool/truncate.ts`, que salva el contenido completo a un fichero temporal, y reducción de resultados históricos durante replay/context building en `session/message-v2.ts`.
- **El runtime repara tool calls defectuosas**: `session/llm.ts` corrige casing cuando puede y deriva llamadas irreparables a la tool `invalid`; los schemas se normalizan en `tool/json-schema.ts` para tolerar diferencias entre proveedores.

## Línea evolutiva reconstruida

La evolución observada se puede resumir así:

```text
tools ad-hoc / registry mutable
        |
        +--> tool schema normalization + plugin/custom tools
        |
        +--> Effect ToolRegistry service + instance-scoped resolution
        |
        +--> explicit ToolPart state machine + result replay guards
        |
        +--> centralized truncation + output metadata

permissions legacy
        |
        +--> permission-rework: allow/deny/ask + pending approvals
        |
        +--> experiments: specificity / ordering / aliases / inherited policy
        |
        +--> dev: ordered last-match rules + Deferred approvals + per-session always rules

bash/shell
        |
        +--> parser/policy hardening + settlement/exit fixes
        +--> external-directory authorization + streamed output/truncation
        `--> canonical shell tool/runtime in dev

patch
        |
        +--> old patch tool
        +--> apply-patch branch: replacement + parser/tests/UI metadata
        +--> symlink/empty/error/optimization hardening
        `--> dev: apply_patch + edit/write model-dependent surface

Code Mode
        |
        +--> execute-code-mode / boundary experiments
        +--> grouped/deferred tool registration
        `--> dev: confined @opencode-ai/codemode interpreter over permission-filtered MCP catalog
```

## Metodología y nivel de certeza

### Confirmado

Se considera **confirmado** lo observado directamente en `dev`, en código de una branch temática concreta o en un diff cuya superficie está suficientemente aislada. Se citan paths, símbolos y, cuando aporta valor, commits/branches.

Ejemplos de evidencia fuerte:

- `packages/opencode/src/tool/tool.ts`
- `packages/opencode/src/tool/registry.ts`
- `packages/opencode/src/tool/truncate.ts`
- `packages/opencode/src/tool/shell.ts`
- `packages/opencode/src/tool/code-mode.ts`
- `packages/opencode/src/tool/apply_patch.ts`
- `packages/opencode/src/tool/edit.ts`
- `packages/opencode/src/tool/external-directory.ts`
- `packages/opencode/src/permission/index.ts`
- `packages/opencode/src/agent/subagent-permissions.ts`
- `packages/opencode/src/session/processor.ts`
- `packages/opencode/src/session/message-v2.ts`
- `packages/opencode/src/session/llm.ts`
- `packages/opencode/src/session/llm/request.ts`

La branch `apply-patch` es especialmente clara: respecto a su merge-base añade `packages/opencode/src/tool/apply_patch.ts`, elimina `packages/opencode/src/tool/patch.ts` y añade tests específicos. La branch `kit/effectify-tool-registry` cambia de forma concentrada `packages/opencode/src/tool/registry.ts`. La branch `nxl/fix-permission-specificity` modifica el algoritmo de selección en `packages/core/src/permission.ts`.

### Inferencia

Algunas topic branches son muy antiguas y han seguido recibiendo cientos o miles de commits. Compararlas directamente con el `dev` de 2026-08-29 produce diffs contaminados por cambios posteriores no relacionados. Para esas ramas se usa:

- nombre y familia temática,
- archivos temáticos todavía presentes,
- changesets/commits de punta cuando son informativos,
- supervivencia o ausencia del concepto en `dev`,
- comparación con branches más pequeñas de la misma familia.

Toda conclusión no demostrable de forma aislada se marca como **inferencia** en los documentos correspondientes.

## Conclusión

El diseño actual es un pipeline de defensa en profundidad: una tool debe ser registrada y visible, sobrevivir al filtrado de permisos, validar su schema, entrar en la máquina de estados de sesión, superar autorizaciones específicas de la propia tool y respetar boundaries de abort/timeout/filesystem. El frontend muestra el estado, pero no es la autoridad de seguridad. La autoridad está en el runtime y en `Permission.Service`.