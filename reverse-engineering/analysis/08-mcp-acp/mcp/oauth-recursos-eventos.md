# MCP: OAuth, resources, eventos y endurecimiento operativo

## Baseline

Este documento usa `dev@dc4449df0d52199704ea4989a5a993ebbc605612` como comportamiento vigente y commits históricos concretos para explicar su evolución.

## OAuth como parte del connection lifecycle

### Hecho confirmado

`packages/opencode/src/mcp/oauth-provider.ts` implementa el contrato `OAuthClientProvider` del SDK MCP. La autenticación no está modelada como un comando aislado: participa en la creación de un client remoto y en la selección del estado de conexión que OpenCode expone.

El provider conserva material necesario para continuar un flow OAuth:

- tokens;
- refresh token y expiración;
- client information obtenido por dynamic client registration;
- PKCE verifier;
- `state`;
- URL del servidor MCP asociada a las credenciales.

`packages/opencode/src/mcp/auth.ts` es el boundary de persistencia de este material.

### Hecho confirmado: aislamiento por URL

Las credenciales se relacionan con el servidor y su URL. La implementación evita tratar dos URLs distintas bajo el mismo nombre como la misma autoridad OAuth.

### Inferencia

Este diseño protege contra reutilización accidental de tokens cuando una entrada de configuración cambia de endpoint, un caso especialmente relevante porque el nombre del MCP es una key elegida por el usuario y no una identidad criptográfica del servidor.

## Estados de autenticación

### Hecho confirmado

El resultado de conexión remoto distingue al menos:

- `connected`;
- `needs_auth`;
- `needs_client_registration`;
- `failed`;
- `disabled` en el modelo completo de disponibilidad.

La branch/commit `refactor/mcp-connection-results@2a25bdce` (`refactor(mcp): use tagged connection results`) formaliza esta distinción mediante un union tagged `connected | unavailable`, en lugar de inferir el resultado comprobando la presencia de `status`.

### Importancia arquitectónica

**Inferencia.** Esta refactorización no cambia únicamente typing: hace explícito que “no conectado” puede ser un estado operativo válido y accionable, no necesariamente una excepción. El servidor puede requerir intervención de usuario antes de que el lifecycle continúe.

## Dynamic client registration y callback

### Hecho confirmado

Cuando el servidor OAuth admite dynamic client registration, OpenCode puede persistir la información del cliente generada y reutilizarla. Si el servidor exige un `client_id` pero no admite registro dinámico, el estado se convierte en `needs_client_registration` y la configuración debe aportar los datos necesarios.

El flow OAuth incluye callback loopback. Los tests de `packages/opencode/test/mcp/oauth-callback.test.ts` validan:

- redirect URI con puerto/path custom;
- finalización y cierre del callback server;
- presentación segura de errores del provider;
- ausencia de puertos fijos en la evolución `4d4eb7de` (`test(mcp): avoid fixed oauth callback ports`).

## Refresh de tokens

### Hecho confirmado

Antes de conectar un servidor remoto, OpenCode intenta refrescar tokens expirados cuando hay refresh token, client information y metadata suficientes.

La evolución `ecf550c8` (`fix(mcp): cancel timed out oauth refresh`) corrige un problema concreto: limitar la Promise por timeout no cancelaba necesariamente la operación de red subyacente. El cambio:

1. crea un `AbortController`;
2. pasa su `signal` a `refreshTokensIfExpired`;
3. propaga el signal a OAuth discovery y `refreshAuthorization`;
4. aborta al finalizar el timeout.

### Inferencia

El objetivo es mantener una propiedad fuerte del lifecycle: un MCP lento o un authorization server colgado no debe retener indefinidamente recursos ni dejar trabajo OAuth huérfano después de que OpenCode haya decidido continuar/fallar.

## Fallback de transport y OAuth

### Hecho confirmado

Para un MCP remoto, OpenCode puede probar Streamable HTTP y luego SSE. Sin embargo, cuando el primer transport produce un error semántico de autenticación, la implementación no sigue probando transportes de forma ciega.

### Inferencia

El transport y OAuth están acoplados en el punto correcto: el fallback sirve para compatibilidad protocolar, no para ocultar un challenge de auth. Esta distinción evita flows duplicados o estados inconsistentes.

## Resources como capability de primera clase

### Hecho confirmado

El commit `e1d1352d` (`feat(mcp): expose resource APIs`) lleva MCP resources a la API/SDK público de OpenCode. Añade operaciones equivalentes a:

- catalogar resources/templates;
- leer un resource por `{server, uri}`;
- representar contenido de texto o blob;
- error tipado cuando el servidor no existe;
- evento `mcp.resources.changed`.

El catálogo agregado conserva:

- `server`;
- `name`;
- `uri` o `uriTemplate`;
- descripción y MIME type cuando el servidor los suministra.

### Boundary HTTP

**Hecho confirmado.** La API pública incorpora rutas equivalentes a:

- `GET /api/mcp/resource`;
- `POST /api/mcp/resource/read`.

Esto permite que TUI/desktop/SDK consuman resources mediante OpenCode, sin abrir una segunda conexión MCP desde cada cliente.

## Resource list change notifications

### Hecho confirmado

`feat/mcp-resource-list-changed@c03c2be4` añade soporte para `ResourceListChangedNotificationSchema` solo cuando `client.getServerCapabilities()?.resources?.listChanged` lo anuncia.

El handler publica `mcp.resources.changed` únicamente si el client que originó la notificación sigue siendo el client activo para ese nombre de servidor. Los tests validan que una notificación tardía de un client reemplazado no invalida el catálogo actual.

### Inferencia

Esta comprobación de identidad es una protección frente a races de reconnect/reconfigure: una conexión antigua puede todavía completar callbacks después de haber sido sustituida.

## Tools changed vs resources changed

### Hecho confirmado

- `tools/list_changed`: provoca re-listado y actualización del cache de tool definitions.
- resource list changed: publica un evento de invalidación; el consumidor vuelve a pedir el catálogo de resources.

### Inferencia

La asimetría es deliberada. Las tools forman parte del catálogo ejecutable del runtime y por eso OpenCode mantiene su cache listo; resources son datos navegables/consultables y pueden mantenerse lazy en el cliente que los necesita.

## TUI y sincronización

### Hecho confirmado

La TUI escucha `mcp.resources.changed` y vuelve a consultar el catálogo. `4d3ff368` (`refactor(tui): simplify MCP catalog refresh`) elimina una capa previa de single-flight/pending-refresh en el cliente y conserva un refresh más directo.

### Inferencia

El cambio sugiere que la consistencia última del catálogo se delega al stream de eventos + lectura autoritativa del servidor OpenCode, en vez de intentar implementar una state machine compleja en cada frontend.

## Entorno de MCP locales

### Hecho confirmado

`restrict-mcp-env@8f626456` (`fix(opencode): make mcp env handling portable`) confirma que los procesos MCP locales no heredan indiscriminadamente todo `process.env`. OpenCode construye un baseline reducido de variables necesarias del sistema y fusiona las variables configuradas explícitamente.

En Windows, la comparación de keys configuradas se hace case-insensitive para variables como `Path`/`PATH`.

### Inferencia

Esto actúa como boundary de seguridad y reproducibilidad: secretos arbitrarios del proceso padre no pasan automáticamente a herramientas MCP externas.

## Timeouts de tool calls

### Hecho confirmado

La branch `elicitation-timeout@b712f3bb` es nominalmente engañosa: su commit es `fix(core): use long mcp tool timeout`. Cambia el timeout de `client.callTool` a un valor efectivamente muy largo y mantiene `resetTimeoutOnProgress`.

### Inferencia

La razón expresada por el código es acomodar tools “human-driven” que pueden bloquear esperando interacción. No es evidencia de ACP elicitation y se clasifica como hardening del runtime MCP.

## Reconnect: qué pertenece a MCP y qué no

### Hecho confirmado

MCP puede reemplazar/recrear clients y sus handlers deben comprobar que siguen siendo activos. Sin embargo, la branch `reconnect-backoff@d72fadea` modifica `packages/client/src/solid/connection.ts`: aplica exponential backoff al **event stream del cliente OpenCode**, no al transport MCP.

### Conclusión

No debe usarse esa branch como prueba de una política de reconexión MCP. Sí es relevante para ACP/TUI en cuanto ambos dependen del flujo de eventos de OpenCode, pero el ownership está en la capa cliente/SDK.

## Eventos MCP relevantes

En la evolución observable aparecen eventos como:

- `mcp.tools.changed`;
- `mcp.resources.changed`;
- eventos relacionados con fallo de apertura del browser OAuth;
- cambios de status observables por clientes/UI.

**Hecho confirmado.** Estos eventos son eventos **OpenCode** derivados de señales MCP. El protocolo externo no atraviesa el sistema sin adaptación.

## Hechos frente a inferencias

### Hechos demostrados

- OAuth persiste estado necesario para refresh/continuación.
- refresh puede ser abortado por timeout.
- callback loopback tiene lifecycle probado.
- resource APIs están expuestas por OpenCode.
- resource list changes respetan capabilities anunciadas.
- handlers antiguos se descartan por identidad de client.
- el entorno de subprocess MCP está restringido.

### Inferencias arquitectónicas

- OpenCode considera MCP un boundary no confiable y stateful.
- el catálogo de resources se trata como estado derivado y refrescable, no como source of truth local.
- los estados tagged de conexión convierten autenticación/configuración en estados de dominio en lugar de errores excepcionales.
