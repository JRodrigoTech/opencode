# ACP: mapping de contenido, attachments, elicitation y conceptos no isomórficos

## Baseline

El comportamiento vigente se compara contra `dev@dc4449df0d52199704ea4989a5a993ebbc605612`. Para elicitation se usa además el commit histórico resoluble `a36c8392b20e306c0688280b9377500a43b486d2`.

## El problema de integración

ACP y OpenCode no utilizan exactamente el mismo modelo para contenido, resources, archivos, tools, formularios o sesiones hijas. El adapter no puede hacer un cast de tipos: necesita una capa explícita de proyección.

### Hecho confirmado

`packages/opencode/src/acp/content.ts` implementa traducciones entre content blocks ACP y prompt/message parts de OpenCode, tanto para entrada como para replay/salida.

## Texto

### Hecho confirmado

Un bloque ACP de texto se transforma en una part textual de OpenCode. En sentido inverso, las parts de texto visibles se reconstruyen como contenido ACP.

El adapter distingue parts sintéticas/ignoradas cuando prepara o reproduce contenido, evitando exponer implementación interna como texto del usuario/agente.

## Imágenes

### Hecho confirmado

ACP image content puede convertirse a una representación de archivo/data URI que OpenCode entiende. En replay, data URIs de imagen pueden volver a proyectarse como image blocks ACP con media type y payload adecuados.

### Inferencia

La capa trata la codificación como transporte, no como identidad: el mismo contenido visual puede atravesar OpenCode como file/data URI y reaparecer como un bloque image ACP.

## Resource links

### Hecho confirmado

ACP `resource_link` se convierte a una referencia utilizable por OpenCode conservando URI, nombre y metadata relevante. Para URIs `file://`, el adapter preserva la semántica de referencia a fichero en lugar de incrustar necesariamente todo el contenido.

### Inferencia

Esta decisión evita inflar prompts y mantiene la identidad de resources direccionables por URI.

## Embedded resources

### Hecho confirmado

El adapter soporta resources ACP cuyo contenido está embebido. Dependiendo del MIME/payload, puede convertirlos a contenido textual o binario/blob y reconstruir la forma ACP durante replay.

### Distinción importante

MCP resources y ACP resources no son el mismo boundary:

- **MCP resource**: vive detrás de un MCP server y se obtiene por catálogo/URI mediante el servicio MCP de OpenCode.
- **ACP resource/resource_link**: forma parte del contenido que un cliente ACP entrega o recibe.

**Inferencia.** Pueden apuntar conceptualmente al mismo material, pero OpenCode no debe confundir su lifecycle: uno pertenece a integración de servidores; el otro, al wire model de una conversación ACP.

## Attachments internos de OpenCode

### Hecho confirmado

OpenCode posee además su propio modelo de file/directory attachments dentro de Session. La branch `prompt-attachments@23751024` solo normaliza paths de attachments de directorio en `packages/core/src/session.ts` y no implementa ACP.

### Conclusión

El nombre de esa branch no debe usarse como evidencia de una generación ACP. Es relevante únicamente porque `acp/content.ts` tiene que interoperar con el modelo de contenido/archivo que el runtime ya posee.

## Elicitation: línea experimental

### Hecho confirmado

El commit `a36c8392` tiene mensaje `feat(cli): support acp elicitation` y añade soporte explícito para `unstable_createElicitation` del SDK ACP.

La implementación:

1. inspecciona `InitializeRequest.clientCapabilities.elicitation`;
2. activa un flag solo si el cliente anuncia form elicitation compatible;
3. intercepta eventos OpenCode `form.created` del turn activo;
4. convierte `FormInfo` a `CreateElicitationRequest`;
5. llama `connection.unstable_createElicitation(...)`;
6. si el cliente acepta, traduce `response.content` a `form.reply` de OpenCode;
7. si rechaza, cancela o la representación no es soportable, llama `form.cancel`.

## Form schema mapping

### Hecho confirmado

La branch experimental traduce fields internos a un schema ACP de tipo object.

Casos observados:

- string → `type: "string"`, con format, min/max length, pattern, default;
- opciones string → `oneOf` con `const`, title y description;
- number/integer → límites y default;
- boolean → default;
- multiselect → array con opciones y min/max items.

Los fields `external` o condicionados mediante `when` no se convierten en esa implementación. Si no puede mapear todos los fields de forma segura, no solicita elicitation y cancela el formulario interno.

### Inferencia

La branch adopta una regla de **all-or-nothing schema fidelity**: es preferible cancelar/fallback que mostrar al usuario un formulario ACP que pierda condiciones o semántica de campos.

## Correlación con tool calls

### Hecho confirmado

El bridge intenta extraer `tool.callID` de `FormInfo.metadata` y lo pasa como `toolCallId` en la petición ACP de elicitation.

### Inferencia

Esto conserva causalidad entre una tool que solicita información y la interacción humana correspondiente, evitando que elicitation aparezca como un diálogo huérfano.

## Estado frente a `dev`

### Hecho confirmado

La implementación `unstable_createElicitation` está demostrada por el commit histórico `a36c8392`. En la baseline `dev` observada, la búsqueda de ese símbolo no devuelve la implementación equivalente.

### Inferencia

Elicitation debe clasificarse como **experimento no consolidado en la baseline analizada**, no como feature ACP vigente. Puede haber sido abandonada, replanteada o pendiente de una API estable del protocolo.

## No confundir con `elicitation-timeout`

### Hecho confirmado

La branch `elicitation-timeout@b712f3bb` no modifica ACP forms. Su commit es `fix(core): use long mcp tool timeout` y solo cambia el timeout de `client.callTool` MCP para admitir herramientas humanas de larga duración.

### Conclusión

El nombre histórico refleja contexto de desarrollo, no el contenido final observable del HEAD/commit.

## Permissions vs elicitation

### Hecho confirmado

Permissions y elicitation son bridges diferentes:

- `permission.asked` → `requestPermission`: autorización de una acción/tool.
- `form.created` → `unstable_createElicitation` en el experimento: obtención de datos estructurados del usuario.

### Inferencia

Separarlos evita convertir toda interacción humana en “permission”. El modelo interno de OpenCode ya distingue consentimiento de captura de datos, y la integración ACP intenta preservar esa diferencia cuando el protocolo lo permite.

## Subagentes: otro mapping no isomórfico

### Hecho confirmado

`acp-subagent-events@ca357ee2` proyecta actividad de child sessions a la sesión ACP raíz y añade metadata privada `opencode/child-session`. También hace únicos los IDs de tool calls al prefijarlos con child session ID.

### Inferencia

La metadata ACP funciona como extension point para conceptos que OpenCode necesita conservar pero que no encajan de forma 1:1 en el modelo base ACP.

## Namespaces y IDs

### ACP

**Hecho confirmado.** IDs de sesión, mensaje, part y tool call se preservan siempre que sea posible. Cuando existe riesgo de colisión entre child sessions, la branch de subagentes compone un ID con contexto de sesión hija.

### MCP

**Hecho confirmado.** MCP usa namespacing del nombre de servidor para incorporar tools de múltiples servidores al catálogo OpenCode.

### Diferencia conceptual

**Inferencia.** El namespace MCP resuelve colisiones entre **proveedores de capacidades**; el prefijo ACP de subagente resuelve colisiones entre **ejecuciones concurrentes/jerárquicas**. Aunque ambas técnicas concatenen IDs, protegen boundaries diferentes.

## Tabla de mapping externo ↔ interno

| Concepto externo | Representación OpenCode | Observación |
|---|---|---|
| ACP text | text part | filtrado de contenido sintético/ignorado |
| ACP image | file/data content | reversible cuando MIME/data URI lo permite |
| ACP resource_link | referencia URI/file | mantiene identidad externa |
| ACP embedded resource | text/blob/file-like part | adaptación por MIME/payload |
| ACP requestPermission | `permission.asked/reply` | opciones once/always/reject |
| ACP elicitation experimental | `form.created/reply/cancel` | solo si capabilities/schema son compatibles |
| ACP child activity experimental | eventos de child Session | proyectados a root + `_meta` |
| ACP MCP server declaration | configuración MCP OpenCode | OpenCode crea/posee la conexión |
| MCP tool | dynamic tool OpenCode | namespaced por servidor |
| MCP resource | catálogo `{server, uri}` | leído a través del servicio MCP |

## Hechos frente a inferencias

### Hechos demostrados

- `acp/content.ts` traduce múltiples tipos de contenido en ambos sentidos.
- MCP resources y ACP content resources atraviesan componentes distintos.
- elicitation fue implementada en un commit histórico con capability negotiation.
- la implementación histórica cancela forms que no puede representar fielmente.
- no hay evidencia del mismo bridge en la baseline `dev` observada.
- la branch `elicitation-timeout` no es ACP elicitation.

### Inferencias arquitectónicas

- ACP actúa como anti-corruption layer y preserva semántica antes que shape idéntico.
- metadata/IDs compuestos son mecanismos deliberados para transportar conceptos internos no nativos de ACP.
- la no consolidación de elicitation probablemente está ligada al carácter `unstable_*` de la API o a diferencias de expresividad entre formularios.
