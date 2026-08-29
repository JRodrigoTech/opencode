# Code Mode (`execute`)

## Objetivo

Code Mode introduce una tool exterior de orquestación, `execute`, que permite al modelo escribir un pequeño programa para encadenar tools conectadas. En `dev` la implementación relevante está en `packages/opencode/src/tool/code-mode.ts` y el intérprete/contrato reusable reside en el paquete `@opencode-ai/codemode`.

## Superficie exterior

**Confirmado:** la tool se identifica como `execute` (`CODE_MODE_TOOL`) y recibe esencialmente código fuente como input. La descripción de la propia tool presenta el entorno como un runtime confinado para orquestar herramientas conectadas, no como ejecución arbitraria del sistema operativo.

La distinción es crítica:

```text
execute != shell
```

El código de `execute` sólo puede alcanzar capabilities que el host haya materializado en su catálogo de child-tools.

## Construcción del catálogo

`code-mode.ts` obtiene tools MCP y las transforma en un árbol namespaceado por servidor.

Flujo reconstruido:

```text
MCP tools
   |
   v
merge agent + session ruleset
   |
   v
Permission.visibleTools(...)
   |
   v
sanitize/group by MCP server
   |
   v
child tool definitions
   |
   v
CodeMode interpreter
```

**Confirmado:** las tools que el ruleset no hace visibles se eliminan **antes** de construir el entorno del intérprete.

Cada child-tool conserva los schemas MCP de input/output y una referencia a la tool MCP original.

## Autorización por child call

El punto de seguridad principal es `invokeChildTool`.

**Confirmado:** cada invocación interna:

1. ejecuta el hook `tool.execute.before`;
2. solicita `ctx.ask()` usando la permission key de la child-tool;
3. sólo tras autorización llama a `client.callTool(...)`;
4. pasa timeout, `AbortSignal` y callback de progreso;
5. valida/normaliza el resultado;
6. ejecuta `tool.execute.after`;
7. actualiza metadata de la invocación.

Por tanto, haber autorizado la tool exterior `execute` no implica autorización transitiva de todo el catálogo interno.

## Estado de child-tools

La metadata del runtime mantiene llamadas hijas con estados conceptuales:

```text
running -> completed
        \-> error
```

Esto permite que una sola tool-part exterior represente un programa con varias acciones internas y que la UI/runtime conozca el progreso sin convertir cada child invocation en una sesión independiente.

## Abort y timeout

**Confirmado:** el `AbortSignal` de `Tool.Context` se propaga al runtime y a `MCP.client.callTool`.

La llamada MCP usa timeout explícito y puede resetearlo con progreso. Esto evita que una child-tool huérfana sobreviva indefinidamente al cancel de la tool exterior.

## Resultados

Code Mode adapta la respuesta MCP al contrato de `Tool.ExecuteResult`.

- texto MCP se concatena/proyecta como output;
- `structuredContent`, cuando existe, tiene prioridad como representación estructurada;
- imágenes/audio y blobs de resource pueden convertirse en attachments de tipo file;
- resources textuales permanecen en la representación textual;
- errores MCP (`isError`) se convierten en fallo de ejecución.

## Boundary de seguridad

La propiedad más importante es que el código interpretado no recibe primitivas generales de Node/Bun por defecto. Su universo de efectos está definido por el objeto de tools que el host entrega.

```text
LLM-generated code
      |
      v
CodeMode interpreter
      |
      +--> only exposed child tools
                    |
                    +--> permission check
                    +--> MCP protocol validation
                    +--> timeout/abort
```

**Inferencia fuerte:** éste es un modelo capability-based. El código no se considera confiable; se limita el conjunto de objetos capaces de producir efectos.

## Relación con permissions

Hay dos filtros acumulativos:

1. `Permission.visibleTools()` reduce el catálogo disponible al programa;
2. `ctx.ask()` autoriza cada llamada concreta.

Esta redundancia es deliberada y coherente con el resto del runtime: visibility y execution authorization son capas distintas.

## Evolución por branches

### `execute-code-mode-v2`

La branch presenta una evolución grande y divergente. En su punta aparece el commit `c590e276398bfd3fbadbf2113144e0bece9bfaa8` con mensaje `feat(core): add grouped and deferred tool registration (#35232)`.

El diff acumulado incluye la introducción de un paquete `packages/codemode/` con:

- `src/codemode.ts`
- `src/tool-runtime.ts`
- `src/tool.ts`
- `src/tool-error.ts`
- `src/token.ts`
- `src/values.ts`
- tests de parity, signatures, enumeration, promises y stdlib.

**Hecho:** esa branch evidencia la extracción de Code Mode a un runtime/package explícito y pruebas de paridad/contrato.

**Precaución:** la branch está muy divergida respecto a `dev`; no es válido atribuir todo su diff acumulado a Code Mode.

### `code-mode-boundary`

**Inferencia respaldada por el estado actual:** representa la fase en la que se endureció la separación entre el intérprete y las capabilities del host. El `dev` actual materializa exactamente ese tipo de boundary: catálogo prefiltrado + child invocations autorizadas individualmente.

### `tools-defer` / grouped and deferred registration

**Inferencia:** estas líneas convergen con el problema de no inicializar/materializar todas las tools indiscriminadamente. El diseño actual del registry y del entorno Code Mode favorece resolución contextual e instance-scoped.

## Diferencias con shell

| Dimensión | Code Mode `execute` | Shell |
|---|---|---|
| Lenguaje ejecutado | DSL/JS confinado del paquete codemode | shell real del sistema |
| Capabilities | catálogo explícito de tools | binarios/comandos del entorno |
| Autorización | cada child-tool | comando/pattern + paths externos |
| Filesystem implícito | sólo a través de child tools | sí, por comandos shell |
| Cancelación | AbortSignal a runtime/MCP | AbortSignal + kill de proceso |
| Boundary principal | capability graph | process + filesystem policy |

## Conclusión

Code Mode no es un bypass de tools ni de permisos. Es un **meta-runtime de orquestación** que cambia la forma de expresar una secuencia de llamadas, pero conserva el control de acceso en el host. La evidencia de `dev` muestra un diseño de capabilities: sólo existen para el programa las child-tools aprobadas para publicación, y cada efecto vuelve a cruzar el boundary de autorización.