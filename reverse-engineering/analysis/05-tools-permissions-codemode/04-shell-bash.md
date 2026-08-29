# Shell / Bash runtime

## Baseline `dev`

La implementación principal reside en `packages/opencode/src/tool/shell.ts`. Aunque la tool se presenta como ejecución de shell, internamente combina parsing, policy derivation, autorización de filesystem, proceso hijo, streaming, truncado, timeout y cancelación.

## Preparación y análisis del comando

**Confirmado:** OpenCode no trata siempre el comando como una cadena opaca. El runtime dispone de parsers para shell compatibles con Bash y PowerShell y analiza la estructura antes de ejecutar.

El análisis extrae información relevante para autorización:

- comandos/programas invocados;
- argumentos significativos;
- paths potencialmente accedidos;
- directorios externos al workspace/instance;
- patrones de permiso reutilizables;
- prefijos que pueden proponerse mediante `always`.

El código de `shell.ts` usa además conocimiento de aridad/comandos para evitar interpretar indistintamente todos los tokens como paths o como comandos.

## Policy antes de spawn

Flujo conceptual:

```text
command string
   |
   v
parse / scan
   |
   +--> command permission patterns
   +--> external directory candidates
   |
   v
ctx.ask(...)
   |
   +--> denied/rejected => no spawn
   `--> allowed
             |
             v
          spawn child
```

La propiedad de seguridad importante es que el proceso no se crea antes de resolver la autorización aplicable.

## Directorios externos

El shell comparte el helper `packages/opencode/src/tool/external-directory.ts` con tools de filesystem.

**Confirmado:** si el scanner concluye que un comando alcanzará una ruta fuera de la instancia, se solicita `external_directory` para el directorio afectado. Esta autorización es independiente del permiso general para ejecutar el comando.

Eso evita una equivalencia peligrosa:

```text
allow shell != allow arbitrary filesystem traversal
```

## Entorno

Antes de ejecutar, el runtime permite que plugins transformen/injecten environment mediante el hook `shell.env`.

**Confirmado:** el proceso recibe un environment ensamblado por el host, no un snapshot completamente opaco del caller. Esto convierte el environment en otra superficie de extensión controlada.

## Ejecución del proceso

La ejecución está conectada al abstraction de process/child-process del runtime Effect.

Propiedades confirmadas:

- cwd determinado por la instancia/contexto;
- captura combinada/gestionada de output;
- `AbortSignal` propagado desde `Tool.Context`;
- timeout configurable, con fallback de aproximadamente dos minutos en el camino normal;
- espera explícita al exit del child;
- kill escalonado cuando hay abort/timeout;
- force-kill posterior si el proceso no termina durante el grace period.

La familia de branches `shell-exit-*` es consistente con la dificultad de este boundary: orden entre output flush, exit status, grace period y settlement del proceso.

## Streaming y metadata

Mientras el proceso corre, `shell.ts` actualiza metadata mediante `ctx.metadata()`.

Esto permite exponer progresivamente:

- preview del output;
- estado de ejecución;
- información del comando;
- truncado/output path cuando se activa.

No se espera al final para persistir toda la información observable.

## Truncado incremental

Shell requiere un tratamiento distinto a una tool que retorna un string pequeño.

**Confirmado:** durante la ejecución se mantiene una ventana de output. Cuando se rebasa el límite:

1. se crea un sink/fichero para el contenido completo;
2. el runtime continúa escribiendo allí mientras llega más output;
3. la metadata conserva una preview acotada;
4. el resultado final devuelve un tail útil y el path al output completo.

Branches temáticas que evidencian esta estabilización:

- `shell-output-preview`
- `shell-progress-tail`
- `shell-tail-output`
- `shell-output-flush`
- `shell-interrupt-output`
- `tool-output`
- `fix-tool-outputs`
- `bound-tool-output`

No todas son cambios del authorization engine; varias se centran en observabilidad, memoria/context budget y orden de flush.

## Abort, timeout y exit

El proceso puede terminar por:

- exit normal;
- error de spawn/process;
- timeout;
- abort del usuario/session;
- interrupción del Effect padre.

La metadata final diferencia estas situaciones cuando es relevante.

La existencia de branches como `shell-exit-flake`, `shell-exit-grace`, `shell-exit-order`, `shell-exit-status` y `shell-spawn-ready` muestra que el lifecycle fue refinado alrededor de races entre:

```text
stdout/stderr completion
process exit
abort signal
timeout
forced kill
```

**Inferencia:** el diseño actual intenta garantizar settlement determinista: no considerar terminada la tool antes de estabilizar output/exit, pero tampoco permitir procesos huérfanos.

## Branch `shell-tool-policy`

Es una branch muy divergente frente al `dev` actual, por lo que el diff completo contiene cambios ajenos. Sin embargo, contiene una evidencia temática explícita: `.changeset/shell-permission-scan.md`.

El changeset indica la adición de un **portable shell permission scanner opt-in** y establece una regla para comandos opacos: usar autorización shell normal sin inferir directorios externos, conservando el path tree-sitter como comportamiento por defecto.

### Interpretación

**Hecho:** existió un esfuerzo por desacoplar la policy de shell de un único parser/plataforma.

**Inferencia:** el objetivo era degradar de forma segura cuando el comando no pudiera analizarse: no inventar paths externos a partir de una parse defectuosa, pero tampoco saltarse la autorización general de shell.

## Branches relacionadas

### Semántica / seguridad

- `bash-tool`
- `shell-tool-policy`
- `shell-tool-fallback`
- `tool/permission`
- `permission-rework`
- `wsl-shell-env`
- `v2-shell-utf8`

### Lifecycle del proceso

- `shell-exit-flake`
- `shell-exit-grace`
- `shell-exit-order`
- `shell-exit-status`
- `shell-spawn-ready`
- `shell-test`
- `shell-tool-test`
- `fix-shell-test`

### Output / streaming

- `shell-interrupt-output`
- `shell-output-flush`
- `shell-output-preview`
- `shell-progress-tail`
- `shell-tail-output`
- `tool-output`
- `fix-tool-outputs`
- `bound-tool-output`

### Presentación/UI

- `shell-badge-error`
- `shell-model-preview`
- `nxl/shell-mode-tray`
- `fix-shell-tab`
- `fix/shell-tab-shell-mode`

Estas últimas no se usan como evidencia del authorization boundary salvo que modifiquen también backend.

## Boundary reconstruido

```text
LLM tool call
    |
    v
Shell parameters
    |
    v
parser/scanner
    |
    +--> shell permission
    +--> external_directory permission
    |
    v
spawn
    |
    +--> shell.env plugin hook
    +--> stream output -> metadata
    +--> truncate/spill to file
    +--> abort / timeout
    |
    v
exit settlement
    |
    v
Tool.ExecuteResult
```

## Conclusión

El shell es una de las tools de mayor confianza requerida y por eso concentra varias capas de defensa: análisis previo, autorización por patrón, permiso separado para filesystem externo, control de proceso, cancelación, timeout y output acotado. Las branches históricas muestran que la mayor parte de la evolución no consistió en “añadir bash”, sino en hacer determinista y seguro el boundary entre una tool call del modelo y un proceso real del sistema.