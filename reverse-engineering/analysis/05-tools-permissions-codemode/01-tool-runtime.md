# Tool runtime

## Baseline `dev`

### Contrato común de tool

El contrato base vive en `packages/opencode/src/tool/tool.ts`.

**Confirmado:** una tool declara:

- `id`
- `description`
- `parameters` mediante `effect/Schema`
- opcionalmente un JSON Schema explícito
- `execute(params, ctx)`

`Tool.Context` transporta los identificadores de sesión/mensaje/call, el agente activo, `AbortSignal`, acceso a metadata incremental y la función `ask()` de permisos. Esto es importante: las tools no conocen el frontend ni resuelven aprobación por sí mismas; consumen una capability de autorización inyectada por el runtime.

`Tool.define()` añade un wrapper uniforme que valida input, instrumenta ejecución y normaliza resultados. El resultado funcional común es `Tool.ExecuteResult`, normalmente con `title`, `metadata`, `output` y, cuando aplica, `attachments`.

### Registry

`packages/opencode/src/tool/registry.ts` es la capa de descubrimiento/materialización.

**Confirmado:** el registry combina tools builtin con extensiones custom/plugins y crea las definiciones en el contexto de instancia. No es únicamente una tabla estática: algunas tools dependen de servicios Effect, flags, plataforma, modelo o configuración.

La branch `kit/effectify-tool-registry` es evidencia histórica fuerte de este boundary: concentra 214 adiciones/139 eliminaciones en `packages/opencode/src/tool/registry.ts` y conecta el servicio al árbol de instancias Effect. La dirección arquitectónica que finalmente aparece en `dev` es separar `ToolRegistry.Service` del caller de sesión.

### Proyección al LLM

`packages/opencode/src/session/llm/request.ts::prepare()` recibe un `Record<string, Tool>` ya materializado y ejecuta `resolveTools()`.

**Confirmado:** antes de enviar tools al proveedor:

1. se fusionan permisos del agente y de la sesión;
2. `Permission.disabled()` identifica tools globalmente deshabilitadas;
3. se respetan overrides `user.tools?.[name] === false`;
4. las tools restantes se ordenan por nombre;
5. determinados proveedores reciben `strict: false` para tolerar schemas dinámicos/MCP;
6. GitHub Copilot puede recibir `_noop` al reanudar histories con tool calls pero sin tools activas.

Esto significa que existe un primer control de acceso por **visibilidad**. Sin embargo, no sustituye el permiso contextual de cada invocación: una tool visible puede pedir `ask()` después según path, comando o subagente concretos.

### Normalización de schemas

`packages/opencode/src/tool/json-schema.ts` convierte Effect Schema a JSON Schema utilizable por proveedores.

**Confirmado:** la capa:

- cachea por objeto Schema (`WeakMap`);
- genera draft 2020-12;
- inlina `$ref` locales cuando puede;
- elimina definitions resueltas;
- normaliza optional/null unions;
- colapsa determinados `anyOf` y `allOf`;
- normaliza integer bounds.

La razón arquitectónica es compatibilidad multi-provider: el schema interno puede ser más expresivo que el subconjunto aceptado por algunos endpoints de tool/function calling.

### Invocación y hooks

La ejecución ordinaria se construye en la sesión. `session/prompt.ts` materializa tools desde `ToolRegistry`, MCP y otras fuentes y entrega adapters al LLM.

**Confirmado:** la ejecución pasa por hooks de plugin `tool.execute.before`/`tool.execute.after` en los paths relevantes. Code Mode también replica explícitamente esos hooks para child-tools MCP.

`session/llm.ts` añade reparación de tool calls: si el modelo usa casing incorrecto pero existe la tool lower-case, corrige el nombre. Si la llamada no puede repararse, transforma la llamada hacia `invalid`, encapsulando el nombre y error originales.

### Máquina de estados persistida

`packages/opencode/src/session/processor.ts` y el schema de `ToolPart` mantienen el lifecycle observable/persistible.

Estados reconstruidos:

```text
pending
  |
  v
running
  | \
  |  \
  v   v
completed error
```

**Confirmado:** la sesión crea/actualiza parts de tipo `tool` con input, timestamps, metadata y output/error. El estado no es decorativo: sirve para replay, interrupciones, UI y reconstrucción del contexto.

Hay paths que crean directamente un estado `running` porque la invocación se inicia desde una acción interna, por ejemplo el manejo de subtasks en `session/prompt.ts`.

### Resultados y attachments

El resultado estándar guarda texto y metadata. Algunas tools devuelven attachments que posteriormente se proyectan a mensajes/model input cuando el provider los soporta.

Code Mode, por ejemplo, convierte bloques MCP `image`, `audio` o resources blob en attachments de tipo `file`, mientras los resources de texto permanecen como contenido textual.

### Truncado operativo

`packages/opencode/src/tool/truncate.ts` define límites globales por defecto:

- `MAX_LINES = 2000`
- `MAX_BYTES = 50 * 1024`

Configurable mediante `tool_output.max_lines` y `tool_output.max_bytes`.

**Confirmado:** si el contenido excede límites:

1. genera un preview `head` o `tail`;
2. escribe el texto completo en el directorio de truncado;
3. devuelve el path mediante `outputPath`;
4. añade instrucciones de recuperación;
5. limpia archivos con más de siete días mediante un proceso periódico.

Si el agente tiene permitido `task`, la pista recomienda delegar el procesado del output grande a un explore agent, evitando cargarlo completo en contexto.

### Shell y truncado incremental

`shell.ts` implementa un caso especial: no espera al final para decidir. Acumula una ventana y, al superar el máximo, crea un fichero/sink de output y continúa escribiendo mientras mantiene metadata preview. Al terminar devuelve el tail y el path completo si hubo corte.

Por tanto hay dos modalidades:

- **post-execution truncation**: `Truncate.output()` para una tool que ya tiene todo el string;
- **stream-aware truncation**: `ShellTool.run()` para procesos potencialmente largos.

### Replay y reducción histórica

`session/message-v2.ts` trata resultados antiguos al reconstruir mensajes para el modelo. Esta reducción es conceptualmente distinta de `Truncate.Service`: su finalidad es evitar que histories antiguas de tool results vuelvan a consumir contexto completo.

También reconoce tool parts interrumpidos/orphaned para no reinterpretarlos como trabajo pendiente en una continuación.

## Boundary reconstruido

```text
ToolRegistry
    |
    v
Tool.Def + Schema
    |
    +---- permission visibility filter ----> provider tool schemas
    |
    v
model emits tool-call
    |
    v
session processor / ToolPart lifecycle
    |
    v
Tool.execute(ctx)
    |
    +--> ctx.ask(...) ------> Permission.Service
    +--> ctx.metadata(...) -> persisted/streamed ToolPart
    +--> AbortSignal
    +--> plugin hooks
    |
    v
Tool.ExecuteResult
    |
    +--> truncation
    +--> attachments
    `--> tool-result -> next LLM turn
```

## Inferencias

- La migración a Effect del registry no parece perseguir sólo estilo funcional; evidencia que el ownership correcto de tools es **instance-scoped** y depende de servicios, no un singleton global mutable.
- La duplicación deliberada de hooks/permissions en Code Mode indica que el boundary de seguridad está definido por **cada side-effecting child invocation**, no por la tool exterior `execute` solamente.
- El doble sistema de truncado muestra que OpenCode distingue explícitamente entre almacenamiento operativo del output y presupuesto de contexto del LLM.