# ACP: arquitectura, lifecycle de agente, sesiones y eventos

## Baseline

Estado vigente comparado contra `dev@dc4449df0d52199704ea4989a5a993ebbc605612`.

## Tesis arquitectónica

### Hecho confirmado

ACP no sustituye el runtime del agente de OpenCode. `packages/opencode/src/acp/agent.ts` implementa la interfaz `Agent` de `@agentclientprotocol/sdk` y delega la mayor parte de la lógica a `packages/opencode/src/acp/service.ts`.

El runtime real continúa perteneciendo a OpenCode:

- sesiones persistentes;
- selección de modelo/variant;
- agentes/modes;
- tools y permisos;
- ejecución del prompt;
- event stream;
- MCP;
- filesystem.

ACP es una **fachada protocolar bidireccional** que convierte requests ACP en operaciones del SDK de OpenCode y eventos internos en `sessionUpdate` ACP.

## Componentes

### `acp/agent.ts`

**Hecho confirmado.** Construye el agent ACP, establece la conexión con el SDK ACP y delega operaciones como initialize/auth/session/prompt/cancel en el servicio.

### `acp/service.ts`

**Hecho confirmado.** Mantiene el estado de adaptación:

- catálogos por cwd;
- sesiones ACP registradas;
- turnos activos/cancelables;
- MCP servers registrados para la sesión;
- model/mode/variant y config options;
- mapping entre IDs ACP e IDs OpenCode cuando es necesario.

### `acp/event.ts`

**Hecho confirmado.** Consume el event stream global de OpenCode y proyecta eventos hacia ACP.

### `acp/permission.ts`

**Hecho confirmado.** Traduce `permission.asked` a `requestPermission`, serializa preguntas por sesión y convierte la respuesta ACP al mecanismo de aprobación/rechazo de OpenCode.

### `acp/content.ts`

**Hecho confirmado.** Es el boundary de traducción de contenido y attachments; se desarrolla por separado en `mapping-attachments-elicitation.md`.

## Initialize y capabilities

### Hecho confirmado

Durante `initialize`, ACP publica metadata del agente y capabilities que OpenCode puede servir a un cliente ACP. El servicio también inspecciona capabilities del cliente para decidir qué features del lado cliente son utilizables.

### Inferencia

El adapter está diseñado como capability-negotiated: una feature ACP no debería asumirse solo porque el SDK la tenga tipada. La branch experimental de elicitation confirma esta filosofía al activar el bridge únicamente cuando el cliente anuncia soporte.

## Autenticación ACP

### Hecho confirmado

ACP expone un método de autenticación de OpenCode (`opencode-login` en la línea observada). El protocolo no reimplementa las credenciales de providers: dirige al mecanismo de login propio de OpenCode.

### Inferencia

Esto preserva un único ownership de credenciales. ACP describe la acción de autenticación al cliente, pero no se convierte en una segunda base de datos de auth de providers.

## Creación de sesión

### Hecho confirmado

`newSession` crea una sesión OpenCode para el cwd solicitado, registra el mapping ACP y devuelve el ID que ACP utilizará como `sessionId`. Durante la preparación se construye el catálogo de configuración válido para ese directorio.

### MCP servers suministrados por ACP

**Hecho confirmado.** La creación/carga de sesión procesa MCP servers recibidos mediante ACP. `ACPService` registra esas definiciones en OpenCode antes o durante el setup de la sesión, de modo que las tools/capabilities MCP quedan disponibles al runtime de OpenCode.

### Inferencia

ACP no actúa como proxy de tool calls MCP. ACP declara integración/configuración; OpenCode establece las conexiones MCP y ejecuta las tools dentro de su propio runtime.

## Load, resume y fork

### `loadSession`

**Hecho confirmado.** Carga una sesión OpenCode existente y reconstruye suficiente estado para que un cliente ACP continúe la conversación. Esto incluye replay de contenido/historial y restauración de opciones cuando pueden inferirse de mensajes previos.

### `resumeSession`

**Hecho confirmado.** Reasocia una sesión persistente al adapter ACP y reconstruye catálogo/configuración antes de continuar.

### `unstable_forkSession`

**Hecho confirmado.** ACP puede mapear el concepto de fork a la capacidad de fork de sesión de OpenCode, generando una nueva sesión relacionada pero independiente para el cliente ACP.

### Inferencia

La existencia separada de load/resume/fork demuestra que ACP no modela una sesión como un socket efímero. La identidad durable está en OpenCode y ACP reconstruye una vista protocolar sobre ella.

## Configuración de sesión

### Hecho confirmado

El servicio genera `configOptions` a partir de catálogos OpenCode, incluyendo opciones relacionadas con model, mode/agent y variants disponibles. Para una sesión cargada puede inspeccionar el historial y recuperar la configuración usada anteriormente.

### Hecho confirmado: `acp-config-commit`

Existe una línea de branch específica orientada a configuración ACP. Se trata como evidencia de que el mapping de configuración evolucionó de forma separada del lifecycle inicial.

### Inferencia

La configuración ACP es una proyección dinámica, no una copia literal de `opencode.json`. Debe respetar tanto el catálogo vigente como los valores efectivos de la sesión.

## Prompt lifecycle

### Entrada

**Hecho confirmado.** Una request `prompt` ACP se transforma a parts de prompt OpenCode y se somete a la sesión correspondiente.

### Turn control

**Hecho confirmado.** El servicio registra control de turnos activos para poder atender `cancel`. La cancelación termina/interrumpe la ejecución de la sesión OpenCode asociada.

### Espera hasta idle

**Hecho confirmado.** El adapter no considera terminado el prompt solo porque la llamada de submit inicial retorne. La lógica de eventos espera la señal de sesión `idle`, garantizando que los chunks/eventos relevantes se hayan proyectado antes de resolver la respuesta final ACP.

### Inferencia

Este detalle soluciona una diferencia semántica entre APIs: el endpoint OpenCode puede disparar una ejecución cuyo progreso llega por stream, mientras ACP espera un lifecycle de request que englobe el turn completo.

## Event stream como backbone

### Hecho confirmado

`acp/event.ts` abre una suscripción al stream global de OpenCode. Entre los eventos traducidos aparecen:

- `message.part.delta`;
- `message.part.updated`;
- `permission.asked`;
- `session.status`;
- eventos de creación/borrado de sesión en la evolución de subagentes.

El adapter mantiene metadata de parts para saber si un delta corresponde a texto visible, reasoning o una tool.

## Texto y reasoning

### Hecho confirmado

Los deltas de texto assistant se convierten en updates ACP de mensaje del agente. Los deltas de reasoning se proyectan como thought/reasoning updates cuando la representación ACP lo permite.

### Inferencia

ACP no recibe eventos internos crudos. La capa usa metadata de mensaje/part para descartar deltas que no deben aparecer al cliente, por ejemplo parts sintéticos/ignorados según el contexto.

## Tool calls

### Hecho confirmado

Cuando un `ToolPart` cambia de estado, ACP genera el lifecycle correspondiente:

- tool call inicial;
- running/update;
- completed;
- error.

La implementación evita emitir múltiples starts para el mismo call y mantiene snapshots para no duplicar output de shell innecesariamente.

### Inferencia

La tool call ACP es una **vista** de la ejecución interna. ACP no controla directamente el state machine de la tool; observa y proyecta el estado autoritativo de OpenCode.

## Permisos

### Hecho confirmado

`ACPPermission.Handler` transforma un `permission.asked` de OpenCode en `requestPermission` ACP con opciones equivalentes a:

- permitir una vez;
- permitir siempre;
- rechazar.

Las peticiones se encadenan por sesión para evitar diálogos concurrentes fuera de orden.

Cuando la conexión ACP no ofrece `requestPermission`, el adapter no puede pedir consentimiento al cliente y debe resolver de forma segura mediante rechazo/fallback interno.

### Escritura de archivos

**Hecho confirmado.** Para ciertos flujos de edición, si el cliente ACP expone `writeTextFile`, el adapter puede usar esa capacidad para reflejar cambios de fichero en el host del cliente.

### Inferencia

Esto revela otro boundary: un cliente ACP puede ser dueño del filesystem visible al usuario, mientras OpenCode sigue siendo dueño del permiso y de la intención de la tool.

## Reconnect del event stream

### Hecho confirmado

La capa de eventos ACP contempla desconexiones del stream y puede volver a suscribirse, manteniendo waiters para conexión e idle.

La branch general `reconnect-backoff@d72fadea` endurece la conexión al event stream en `packages/client`, aplicando exponential backoff hasta 30 s. No es un cambio ACP específico, pero afecta a consumidores del stream, incluido el patrón del adapter.

## Event ordering y replay

### Hecho confirmado

Para sesiones cargadas, el adapter hace replay de mensajes existentes antes de depender exclusivamente de los nuevos eventos. Durante un turn activo, usa el orden del stream para emitir chunks y espera idle antes de completar.

### Inferencia

Esta combinación replay + live stream es necesaria para dar a ACP una vista coherente sin hacer que ACP sea el source of truth del historial.

## Subagentes y sesiones hijas

### Hecho confirmado

La branch `acp-subagent-events@ca357ee2` (`fix(acp): surface subagent activity`) añade un registro de child sessions con:

- `id`;
- `parentID`;
- `rootID`;
- `depth`;
- `title` opcional.

Los eventos de una child session se resuelven hacia la sesión ACP raíz. Los updates reciben metadata `_meta["opencode/child-session"]`; tool call IDs pueden prefijarse con el ID hijo para evitar colisiones.

Permisos de la child session también se proyectan hacia la raíz con contexto adicional.

### Inferencia

OpenCode y ACP tienen granularidades distintas: OpenCode expresa un subagente como una sesión real; el cliente ACP puede estar conectado a una única sesión lógica. La branch propone conservar esa sesión raíz y transportar la jerarquía como metadata en vez de crear sesiones ACP espontáneas.

## Línea histórica `nxl-acp-*`

### Hecho confirmado

`nxl-acp-v1` y `nxl-acp-lifecycle` comparten el mismo merge-base histórico con `dev`; `v1` contiene un commit propio y `lifecycle` dos commits propios respecto a ese punto. Por ello forman una sola línea incremental, no dos arquitecturas independientes.

### Inferencia

La secuencia representa:

1. primera fachada Agent/Service ACP;
2. endurecimiento del lifecycle, eventos y permisos;
3. experimentos posteriores como elicitation/config/subagent projection.

## Mapping resumido

| ACP | OpenCode |
|---|---|
| Agent | fachada sobre `ACPService` |
| sessionId | sesión persistente OpenCode registrada por el adapter |
| new/load/resume/fork | operaciones del subsistema Session |
| prompt | submit de parts a Session + espera por event stream |
| cancel | interrupt/cancel de ejecución OpenCode |
| sessionUpdate | proyección de eventos OpenCode |
| tool call | proyección de `ToolPart` |
| requestPermission | bridge desde `permission.asked` |
| configOptions | proyección de catálogo model/agent/variant |
| MCP servers | configuración registrada en el servicio MCP de OpenCode |

## Hechos frente a inferencias

### Hechos demostrados

- ACP es implementado por Agent + Service + Event + Permission + Content.
- las sesiones siguen siendo sesiones OpenCode persistentes;
- el stream global alimenta los updates ACP;
- prompt completion está coordinado con `idle`;
- permisos y tool states son traducidos;
- MCP servers ACP se registran en OpenCode;
- la branch de subagentes proyecta child sessions hacia una root ACP.

### Inferencias arquitectónicas

- ACP está diseñado como anti-corruption layer entre dos modelos de runtime.
- OpenCode conserva ownership de estado y ACP conserva ownership del wire contract.
- metadata extensible es la vía escogida para representar conceptos OpenCode que ACP no modela nativamente con la misma granularidad.
