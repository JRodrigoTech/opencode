# Patch, edit y mutaciones de archivos

## Evolución principal: `patch` -> `apply_patch`

La branch histórica `apply-patch` ofrece una evidencia especialmente limpia de la sustitución de la herramienta antigua.

Comparada con su línea histórica, esa branch:

- añade `packages/opencode/src/tool/apply_patch.ts`;
- elimina `packages/opencode/src/tool/patch.ts`;
- añade una batería extensa de tests `packages/opencode/test/tool/apply_patch.test.ts`;
- elimina los tests del antiguo `patch`;
- modifica el registry para exponer la nueva tool.

**Hecho:** `apply_patch` no es sólo un rename visual; representa la nueva implementación de patch y reemplaza explícitamente a `patch`.

## `apply_patch` vigente

La implementación actual está en `packages/opencode/src/tool/apply_patch.ts`.

### Fase 1: parse y validación

El input principal es `patchText`.

Antes de mutar filesystem, la tool:

1. valida que exista contenido;
2. parsea el formato patch;
3. rechaza patch vacío/sin hunks útiles;
4. resuelve paths respecto a la instancia;
5. determina operaciones add/update/delete/move;
6. carga el estado previo necesario;
7. calcula el contenido final y diffs.

Para updates verifica además que el target exista y no sea un directorio.

### Fase 2: boundaries de filesystem

Cada target se somete a `assertExternalDirectoryEffect()` cuando corresponde.

En movimientos, el destino también constituye un boundary independiente. Por tanto, un patch que mueve un fichero fuera del worktree no queda cubierto únicamente por la autorización del path origen.

### Fase 3: precomputar el cambio

**Confirmado:** la tool calcula previamente:

- contenido anterior/nuevo;
- diff;
- additions/deletions;
- metadata por archivo;
- detalles necesarios para el approval.

La autorización se solicita con permission `edit` y patrones basados en los paths afectados.

Esto permite presentar al usuario el cambio antes de escribirlo.

### Fase 4: authorize

```text
parsed operations
      |
      v
resolved target files
      |
      v
external_directory checks
      |
      v
computed diffs
      |
      v
ctx.ask(permission="edit", patterns=files)
```

Si existe deny o rechazo, no se ejecuta la fase de mutación.

### Fase 5: mutate

Sólo después de aprobar:

- se escriben archivos nuevos/actualizados;
- se eliminan targets delete;
- se realizan moves;
- se preserva/gestiona BOM cuando aplica;
- se ejecuta formatting cuando corresponde;
- se publican eventos de filesystem/watcher;
- se toca LSP y se recuperan diagnostics.

### Fase 6: resultado

El resultado incluye metadata de files/diffs y puede incorporar diagnostics posteriores a la mutación.

El patrón completo es:

```text
VERIFY -> DESCRIBE -> AUTHORIZE -> MUTATE -> OBSERVE
```

## `edit`

`packages/opencode/src/tool/edit.ts` cubre una mutación puntual por sustitución textual.

### Contrato

Input:

- `filePath`
- `oldString`
- `newString`
- `replaceAll?`

### Safety properties confirmadas

- requiere path válido;
- rechaza `oldString === newString`;
- protege el path mediante semaphore por fichero para evitar writes concurrentes en el mismo target;
- autoriza `external_directory` si procede;
- calcula diff antes de `ctx.ask(permission="edit")`;
- sólo escribe después de aprobación;
- preserva line endings y BOM;
- ejecuta formatter;
- publica eventos;
- solicita diagnostics al LSP.

Para creación de un fichero nuevo mediante `oldString === ""`, sólo acepta el caso si el fichero todavía no existe; evita utilizar `edit` como full-file overwrite accidental de un target existente.

## `write` y superficie de edición

Aunque este documento se concentra en `edit`/`apply_patch`, el sistema de permisos agrupa varias tools bajo la capability conceptual `edit`.

El código de permission contiene una lista equivalente a `edit`, `write` y `apply_patch` para decisiones como tool disabling. Esto muestra que los nombres concretos de tool y el nombre lógico del permiso no siempre son 1:1.

## Branches de hardening de patch

El inventario contiene:

- `apply-patch`
- `patch-empty-hang`
- `patch-error-style`
- `patch-input-guidance`
- `patch-symlink-policy`
- `kit/collapse-patch`
- `optimize-apply-patch`

### `patch-empty-hang`

**Inferencia fuerte por la línea final:** corresponde a hardening frente a patches degenerados/inputs que podían no progresar. El `dev` actual rechaza inputs vacíos/sin operaciones útiles antes de mutar.

### `patch-symlink-policy`

La branch está altamente divergida respecto a `dev`, por lo que no es fiable atribuir todo el diff acumulado. Su existencia indica una fase explícita de revisión del tratamiento de symlinks/path policy.

**Hecho vigente:** los paths pasan hoy por resolución de instancia y por `external_directory`; las operaciones se validan antes del write.

**Inferencia:** la branch de symlink policy forma parte del endurecimiento contra escapes o discrepancias entre paths lógicos y targets reales.

### `patch-error-style` / `patch-input-guidance`

Estas líneas son principalmente ergonomía/protocolo de error e instrucciones al modelo. Son relevantes porque el tool runtime necesita que los errores sean reparables por el agente, pero no constituyen por sí solas un nuevo security boundary.

### `optimize-apply-patch`

**Inferencia:** optimización sobre la implementación ya estabilizada; no altera el modelo conceptual verify/authorize/mutate salvo evidencia específica en commits aislados.

## Relación con snapshots, watcher y LSP

Las tools de edición no terminan su responsabilidad en `writeFile`.

`edit` y `apply_patch` producen información que otros subsistemas utilizan:

- diffs/snapshots para review/revert;
- eventos de FileSystem/Watcher para sincronización;
- LSP `touchFile` y diagnostics para feedback inmediato;
- metadata de tool para UI y history.

Esto revela que el boundary de una mutación de archivo cruza varias capas, pero el **punto de autorización debe ocurrir antes de la mutación** y no después de que esos consumidores reciban el evento.

## Riesgos mitigados

### Mutación fuera del workspace

Mitigada mediante `external_directory`.

### Aprobación ciega

Mitigada precomputando diff y metadata antes de pedir `edit`.

### Races de edición

`edit` serializa acceso por path mediante semaphore.

### Cambio accidental completo

`edit` restringe el uso de `oldString === ""` sobre archivos existentes.

### Estado secundario obsoleto

Después de mutar se notifican watcher/LSP/event bus.

## Inferencias arquitectónicas

- `edit` es la capability de autorización; `apply_patch` y `write` son distintos protocolos para producir una mutación bajo esa capability.
- El sistema prioriza autorización sobre **intención materializada**: pide permiso después de calcular qué archivos/diff resultarán, no sólo sobre el texto crudo generado por el modelo.
- La sustitución de `patch` por `apply_patch` y las posteriores branches de hardening sugieren que el parser/applier se considera una boundary de seguridad y consistencia, no una mera utilidad de texto.