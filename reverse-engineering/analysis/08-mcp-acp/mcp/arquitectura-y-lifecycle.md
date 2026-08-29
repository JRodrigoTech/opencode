# MCP: arquitectura y lifecycle

## Baseline

Estado observado en `dev@dc4449df0d52199704ea4989a5a993ebbc605612`.

## Arquitectura vigente

### Hecho confirmado

El núcleo MCP vive en `packages/opencode/src/mcp/`. La pieza central es `index.ts`, que expone el servicio MCP como capa Effect y mantiene estado asociado a la instancia/proyecto de OpenCode. El servicio no es un simple helper stateless: administra clientes MCP conectados, resultados de conexión, credenciales OAuth asociadas, cache de tools e invalidación por eventos del protocolo.

Piezas principales:

- `index.ts`: lifecycle de conexión, desconexión, creación de clients, status y fachada pública.
- `catalog.ts`: paginación y normalización de tools, prompts, resources y templates.
- `oauth-provider.ts`: implementación de `OAuthClientProvider` del SDK MCP.
- `auth.ts`: persistencia de material OAuth por servidor/URL.
- `server/routes/mcp.ts`: proyección del subsistema hacia HTTP/API.

### Inferencia

El boundary real de MCP está en la capa de integración, no en el tool runtime. El tool runtime consume definiciones ya transformadas; MCP es responsable de conexión, discovery, identidad del servidor, auth y conversión del protocolo externo a conceptos OpenCode.

## Tipos de servidor y transportes

### Local

**Hecho confirmado.** Los servidores locales se ejecutan con `StdioClientTransport`. Su configuración contiene comando, argumentos y entorno. El lifecycle del child process queda encapsulado por el transport MCP; al liberar la capa se cierran clientes/transports.

### Remoto

**Hecho confirmado.** Los servidores remotos prueban transportes en orden práctico:

1. Streamable HTTP.
2. SSE como fallback.

Los headers configurados se aplican también a OAuth discovery/refresh mediante wrappers de `fetch`.

### Razón del fallback

**Inferencia fuerte.** El orden refleja la transición del estándar MCP desde SSE hacia Streamable HTTP sin romper compatibilidad con servidores previos. El código no exige a OpenCode conocer de antemano qué generación de transport implementa cada endpoint.

## State machine de conexión

### Hecho confirmado

La capa distingue estados semánticos de servidor, incluyendo conectividad y autenticación. A nivel externo aparecen al menos:

- `connected`
- `failed`
- `needs_auth`
- `needs_client_registration`

Los errores OAuth no se tratan como un fallo genérico de red: cortan el fallback de transport cuando la acción correcta es pedir autenticación o registro de cliente.

### Inferencia

Esta separación es arquitectónicamente importante porque evita que el fallback esconda un challenge OAuth real. Si Streamable HTTP responde de forma compatible pero requiere auth, probar SSE sería conceptualmente incorrecto.

## Connection lifecycle

### Inicialización

**Hecho confirmado.** Al construir la capa se lee configuración MCP y se preparan conexiones por servidor habilitado. Cada conexión produce un resultado independiente; el fallo de un servidor no invalida necesariamente el subsistema completo.

### Discovery

**Hecho confirmado.** Tras conectar, el runtime materializa el catálogo de tools y registra listeners para `tools/list_changed`. Prompts/resources se consultan mediante funciones de catálogo cuando son necesarios.

### Refresh

**Hecho confirmado.** Cuando el servidor emite cambios relevantes, OpenCode invalida/refresca la vista correspondiente. La evolución posterior añadió eventos específicos para resources.

### Teardown

**Hecho confirmado.** El servicio cierra clients/transports cuando su scope termina. Esta semántica de acquire/release es una diferencia importante respecto a implementaciones tempranas más ad-hoc.

## Tool discovery y adaptación

### Hecho confirmado

`McpCatalog` lista tools mediante el SDK MCP, siguiendo cursores cuando el servidor pagina resultados. Cada tool se transforma a una definición consumible por OpenCode con:

- nombre namespaced;
- descripción;
- input schema;
- wrapper de ejecución que invoca `callTool` en el client MCP correspondiente;
- normalización tolerante de output schemas defectuosos en evoluciones posteriores.

### Namespacing

**Hecho confirmado.** El nombre interno combina servidor y tool, sanitizando caracteres para producir una key estable, conceptualmente:

`<server>_<tool>`

### Inferencia

El namespace no es solo presentación. Funciona como mecanismo de aislamiento en un catálogo único de tools que puede mezclar múltiples servidores externos y tools internas.

## Prompts

### Hecho confirmado

OpenCode puede listar prompts y obtener un prompt concreto de un servidor MCP. La respuesta se proyecta a estructuras internas/SDK sin convertir el prompt en una tool.

### Evolución histórica

La branch histórica `mcp-prompts` proviene de la línea `v2` y muestra una etapa donde prompts se incorporaban al discovery de manera más temprana/eager. El HEAD contiene merges generales, por lo que el propósito se identifica por commits y paths MCP, no por el diff completo contra `dev`.

## Resources y resource templates

### Hecho confirmado

El catálogo vigente soporta:

- `listResources`
- `listResourceTemplates`
- `readResource`

Cada entrada conserva el servidor de origen. Las URIs pertenecen al servidor MCP, mientras OpenCode añade identidad de servidor en el catálogo agregado.

### Cambio de diseño importante

El commit `e1d1352d` (`feat(mcp): expose resource APIs`) elevó resources a API pública de OpenCode/SDK, añadiendo catálogo, lectura y evento `mcp.resources.changed`. Esto demuestra que resources dejaron de ser un detalle interno del agente/TUI para convertirse en capacidad de plataforma.

## Paginación y tolerancia

### Hecho confirmado

`catalog.ts` abstrae la paginación de respuestas MCP. Esto evita duplicar lógica de cursor para tools/prompts/resources/templates.

Las ramas de endurecimiento, por ejemplo `kit/mcp-tolerate-bad-output-schemas`, muestran que OpenCode optó por tolerar servidores parcialmente incompatibles en vez de hacer fallar todo el catálogo por schemas de output defectuosos.

### Inferencia

La filosofía de compatibilidad es deliberadamente defensiva: el protocolo externo se considera un boundary no confiable y se normaliza antes de entrar al runtime interno.

## Tool cache e invalidación

### Hecho confirmado

El catálogo de tools se cachea por conexión/servidor. La notificación MCP `tools/list_changed` invalida la representación y dispara el refresh necesario.

### Inferencia

El cache reduce roundtrips de discovery en cada prompt, pero obliga a OpenCode a modelar notificaciones de capability change como parte del lifecycle. Esto es una razón por la que MCP terminó convirtiéndose en servicio stateful.

## Relación con el servidor HTTP de OpenCode

### Hecho confirmado

`packages/opencode/src/server/routes/mcp.ts` expone operaciones del subsistema MCP sobre la API de OpenCode. En la evolución observada se incorporaron endpoints para listar servidores y, posteriormente, catalogar/leer resources.

### Inferencia

Este boundary permite que TUI, desktop, ACP u otros clientes no tengan que hablar MCP directamente. Hablan con OpenCode, y OpenCode conserva ownership de transport, OAuth y compatibilidad protocolar.

## Comparación con la generación temprana

### Hecho confirmado

Las branches tempranas basadas en `v2` contienen implementación bajo `packages/core/src/mcp/index.ts`. Esa generación concentraba más discovery y orchestration en una única unidad y estaba más cerca de un cliente MCP acoplado al core.

### Inferencia

La evolución hacia `packages/opencode/src/mcp/` + `catalog.ts` + auth/provider + rutas API refleja una separación progresiva de cuatro concerns:

1. connection lifecycle;
2. auth;
3. catalog/protocol adaptation;
4. exposición a clientes internos/externos.

## Qué llegó a `dev`

- multi-transport remoto;
- stdio local;
- OAuth persistente;
- refresh antes de conectar;
- states semánticos de auth;
- tool discovery namespaced;
- prompts;
- resources y templates;
- resource APIs;
- eventos de cambio;
- teardown lifecycle-aware;
- tolerancia a servidores defectuosos.

## Qué cambió o se descartó

### Hecho confirmado

El discovery eager generalizado de la era temprana no es la forma dominante en `dev`: tools mantienen cache/invalidation explícita, mientras prompts/resources se consultan bajo demanda.

### Inferencia

El cambio reduce coste inicial y acoplamiento, y evita que una capability secundaria defectuosa impida conectar un servidor cuyas tools sí son válidas.
