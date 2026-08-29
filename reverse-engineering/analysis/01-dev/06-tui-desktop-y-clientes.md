# 06 — TUI, Desktop y clientes

## Principio común

**[CONFIRMADO]** Las principales interfaces de usuario no son dueñas del runtime del agente. TUI y Desktop consumen un backend mediante contratos de red/API aunque ese backend se ejecute en el mismo proceso lógico, en un Worker o como sidecar local.

**[INFERENCIA]** Esta separación permite reutilizar el mismo runtime para terminal, desktop, web, automatización y clientes externos, y reduce el acoplamiento entre lifecycle visual y lifecycle de sesiones.

## TUI

### Composition

Paths:

- `packages/opencode/src/cli/cmd/tui.ts`
- `packages/opencode/src/cli/tui/worker.ts`
- `packages/tui/src/index.tsx`
- `packages/tui/src/app.tsx`

`@opencode-ai/tui` usa Solid/OpenTUI y monta providers de SDK, proyecto, sync, datos, permisos y otras preocupaciones de presentación.

**[CONFIRMADO]** `TuiInput` recibe URL, argumentos, configuración, `fetch`, headers, source de eventos, plugin host y directorio. Esa firma hace explícito que el frontend puede operar contra transports distintos.

### Backend en Worker

La TUI normal crea un Worker que importa el server y los servicios de instance. El Worker:

- ejecuta `Server.Default().app.fetch` para requests internos;
- retransmite eventos globales por RPC;
- puede arrancar un listener TCP real;
- invalida Config y dispone instances en reload;
- dispone instances y server durante shutdown.

**[CONFIRMADO]** Esto aísla el backend del render loop terminal sin introducir un protocolo funcional alternativo.

### Internal fetch

En modo local normal:

```text
TUI
  -> SDK/fetch
  -> RPC al Worker
  -> Server.Default().app.fetch
  -> handlers/runtime
```

No se abre un puerto y se usa la URL lógica `http://opencode.internal`.

### Modo de red

Con opciones de red, el Worker llama `Server.listen`, la TUI usa una URL real y adjunta autenticación.

### Mini mode

`TuiThreadCommand --mini` delega en `cli/cmd/run` (`runMini`) y tiene restricciones/opciones propias de replay/demo.

**[CONFIRMADO]** Por tanto existen dos superficies terminales con caminos de presentación distintos. No deben confundirse al estudiar lifecycle o replay.

## Estado de UI

**[CONFIRMADO]** La TUI posee stores/providers de presentación y sincronización, pero las entidades autoritativas de sesión/mensaje/tool state están en el backend. Eventos y consultas SDK actualizan la representación local.

**[INFERENCIA]** La UI sigue un patrón cercano a replicated/read-model state: mantiene una copia optimizada para render que puede reconstruirse desde backend + eventos.

## Desktop

### Host Electron

`packages/desktop/src/main/index.ts` controla lifecycle de aplicación, ventanas, IPC, updater, logging, deep links y backend local.

El source tree está separado en:

- `src/main`;
- `src/preload`;
- `src/renderer`.

Esto preserva el boundary de seguridad habitual de Electron entre renderer y capacidades nativas.

### Sidecar backend

**[CONFIRMADO]** Desktop espera explícitamente a que un backend local esté listo y comunica al renderer `url`, `username` y `password`.

Camino V1:

1. elige loopback y un puerto disponible;
2. genera password aleatorio;
3. ejecuta `spawnLocalServer`;
4. espera health/readiness;
5. entrega credenciales al renderer.

Camino V2:

- `SIDECAR_VERSION` depende de `OPENCODE_SIDECAR_V2`;
- con valor `1`, usa `startBackgroundCli` y recibe URL/credenciales del nuevo sidecar.

**[CONFIRMADO]** Desktop contiene así una migración de lifecycle de backend V1/V2 separada de la UI.

### Seguridad y red local

El proceso main:

- asegura bypass de proxy para loopback;
- configura certificados/proxy del sistema;
- usa credenciales para el server local;
- mata sidecars en quit/relaunch;
- deshabilita la web UI embebida del sidecar mediante `OPENCODE_DISABLE_EMBEDDED_WEB_UI`.

**[INFERENCIA]** El sidecar se trata como servicio local autónomo y no como detalle interno del renderer.

### WSL

**[CONFIRMADO]** En Windows existe un controlador separado para sidecars WSL y handlers IPC específicos. Esto extiende el mismo modelo de backend remoto/local a otro entorno de ejecución sin cambiar el renderer en su esencia.

## SDK como frontera cliente

La TUI importa tipos/eventos de `@opencode-ai/sdk/v2` y monta un `SDKProvider`.

**[INFERENCIA]** El SDK es la representación programática del contrato server-cliente y constituye un buen punto de observación para reconstruir APIs públicas sin depender de detalles internos del application runtime.

## Diferencia de ownership

| Responsabilidad | Backend | TUI/Desktop |
|---|---:|---:|
| Ejecutar SessionPrompt | sí | no |
| Resolver tools/provider | sí | no |
| Persistir mensajes | sí | no |
| Evaluar policy/permission | sí | presenta/contesta prompts |
| Mantener PTY | backend/server | presenta/transporta I/O |
| Render conversation | no | sí |
| Estado de ventanas/tema/layout | no | sí |
| Lifecycle sidecar | server/host | Desktop main supervisa |

## Consecuencia arquitectónica

**[INFERENCIA]** La arquitectura vigente permite pensar en TUI y Desktop como **shells de producto** alrededor de un servicio de agente común. La frontera decisiva no es “CLI vs GUI”, sino “cliente vs backend de instance/session”.