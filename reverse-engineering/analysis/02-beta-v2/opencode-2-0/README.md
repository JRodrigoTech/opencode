# Branch `opencode-2-0`: portabilidad Node y desacoplamiento de Bun

## Clasificación

`opencode-2-0` fue una branch de refactor transversal anterior a la V2 madura. Su propósito principal era reducir dependencias directas del runtime Bun y preparar OpenCode para ejecutarse y distribuirse sobre Node.js, especialmente en servidor, procesos, instalaciones npm, OAuth, PTY y entornos Windows/CI.

La branch completa no se integró como una unidad; sus cambios fueron extraídos deliberadamente a PRs menores que sí llegaron a `dev`.

## Hechos demostrables

### Branch y PR principal

- HEAD analizado: `3eeeec359acda9f14334cb420af84f31bee95f11`.
- Fecha del HEAD: 2026-03-20.
- PR upstream: #16918 `opencode 2-0`.
- Base del PR: `dev`.
- El PR #16918 terminó cerrado sin merge como unidad.

El body de #16918 enumera explícitamente, entre otros:

- `core: add Node.js runtime support`;
- `refactor(server): replace Bun serve with Hono node adapters`;
- bundle de database migrations para Node;
- cambios en npm/package installation;
- correcciones Windows y sandbox/CI;
- custom tool/module path loading;
- server lifecycle y `stop()`;
- eliminación de shell execution/server URL del plugin API.

## El dato arqueológico clave: la mega-branch fue descompuesta

El historial de `opencode-2-0` termina con commits del tipo:

```text
chore: extract node entry point into #18324
chore: extract OAuth changes into #18327
chore: extract misc fixes into #18328
```

Más importante aún, el PR merged #18335 declara explícitamente:

> This is the server layer portion remaining from the `opencode-2-0` branch after extracting SQLite abstraction, portable process, OAuth, node entry point, and other changes into separate PRs (#18308, #18318, #18320, #18324, #18327, #18328).

Por tanto, la relación evolutiva no es una hipótesis: `opencode-2-0` actuó como branch integradora/prototipo y después fue repartida en cambios consumibles por `dev`.

## 1. Package installation: de BunProc a npm/Arborist

### PR #18308 — merged

Título:

```text
refactor: replace BunProc with Npm module using @npmcli/arborist
```

Cambios declarados:

- elimina `src/bun/` (`BunProc` + `PackageRegistry`);
- sustituye `bun add/install/info`;
- crea `src/npm/index.ts`;
- usa `@npmcli/arborist`;
- actualiza config, formatter, LSP, plugins y provider loading.

### Significado arquitectónico

**HECHO.** La instalación dinámica de dependencias deja de estar ligada al ejecutable Bun.

**INFERENCIA fuerte.** Este refactor convierte npm/package resolution en un servicio de infraestructura independiente del runtime que ejecuta OpenCode, prerrequisito para un verdadero target Node.

## 2. Process abstraction

### PR #18318 — merged

Título:

```text
refactor: replace Bun shell execution with portable Process utilities
```

Cambios:

- `Bun.spawn` en MCP -> `Process.lines()`;
- `Bun.$` en shell substitutions de prompts -> `Process.text()`;
- elimina dependencias directas de APIs Bun en esos paths.

### Significado

La abstracción `Process` se convierte en boundary para spawn/shell, de modo que MCP y session prompt no necesitan conocer Bun.

Esta decisión es consistente con la V2 posterior, donde el runtime introduce implementaciones por plataforma y trata filesystem/process/PTY como servicios de infraestructura.

## 3. Executable discovery

### PR #18320 — merged

Título:

```text
fix: include cache bin directory in which() lookups
```

- mueve/integra el bin cache en la resolución;
- `which()` incorpora `Global.Path.bin` al PATH efectivo;
- permite localizar tools instalados dinámicamente.

Aunque es un cambio menor, forma parte de la misma cadena: instalación npm portable sólo funciona si executables instalados pueden resolverse sin asumir el comportamiento de Bun.

## 4. Node entrypoint y build

### PR #18324 — merged

Título:

```text
feat: add Node.js entry point and build script
```

El PR añade:

- `src/node.ts`;
- `script/build-node.ts`;
- target de build Node;
- database migrations embebidas durante build;
- un entrypoint que inicia el server.

El propio body declara que depende del server refactor Hono/Node para que `Server.listen()` funcione en Node.

### Boundary descubierto

Para soportar Node no bastaba con recompilar el CLI: había que separar como mínimo:

```text
runtime entrypoint
server transport
DB migration packaging
process execution
package installation
OAuth callback HTTP
PTY/WebSocket behavior
```

Esto revela cuánto del antiguo runtime asumía Bun de forma transversal.

## 5. OAuth HTTP callbacks

### PR #18327 — merged

Título:

```text
refactor: replace Bun.serve with Node http.createServer in OAuth handlers
```

Se sustituyen servidores `Bun.serve` en:

- MCP OAuth callback;
- Codex plugin OAuth callback.

por `node:http.createServer`.

### Significado

**HECHO.** OAuth era otro punto de dependencia Bun que no pertenecía al main server.

**INFERENCIA.** La arqueología de esta branch muestra que “portar el server” no era suficiente: pequeños servidores auxiliares también debían migrarse para que el proceso completo fuera realmente portable.

## 6. Server: Hono Node adapters

### PR #18335 — merged

Título:

```text
refactor(server): replace Bun serve with Hono node adapters
```

Propósito declarado:

- sustituir `Bun.serve` por `@hono/node-server`;
- usar `@hono/node-ws` para WebSocket;
- adaptar PTY route upgrade;
- hacer `Server.listen()` async;
- devolver listener estructurado con `hostname`, `port`, URL y `stop()`;
- adaptar workspace server lifecycle.

### Arquitectura antes

```text
Hono app
   |
Bun.serve
   |
Bun WebSocket API
```

### Arquitectura propuesta

```text
Hono app
   |
Hono adapter boundary
   +--> Node HTTP server
   +--> Node WebSocket adapter
```

El cambio desacopla el router HTTP del servidor concreto.

### Relación con `dev`

#18335 está merged y `dev` posterior incorpora esta línea de portabilidad. Por tanto, éste es uno de los casos más claros donde una idea de `opencode-2-0` llegó posteriormente a `dev` mediante extracción.

## 7. Windows / custom tools / misc

### PR #18328 — merged

Incluye, entre otros:

- workaround para dynamic import de custom tool paths en Windows;
- pequeños ajustes de server/project;
- cambios de process invocation.

La mega-branch original también contenía múltiples fixes relacionados con:

- symlink/bin links;
- npm packages en Windows;
- proxies/CI;
- formatter executable paths;
- plugin module paths.

**INFERENCIA fuerte.** La portabilidad Node se estaba usando también como vehículo para eliminar supuestos Unix/Bun implícitos que dificultaban Windows y ambientes sandboxed.

## 8. Agent runtime

`opencode-2-0` no propone un nuevo agent state machine comparable a la V2 posterior.

Los cambios que afectan agent execution son principalmente infrastructurales:

- shell/process execution portable;
- package/tool availability;
- server availability;
- plugin API cleanup.

Por tanto:

**HECHO.** No existe evidencia en esta branch de una reescritura completa del agent orchestration model.

## 9. AI stack y providers

La branch toca providers indirectamente al cambiar instalación dinámica y runtime dependencies, pero no introduce la abstracción schema-first `@opencode-ai/ai` de la V2 madura.

La principal preocupación provider-related aquí es que provider/plugin packages puedan instalarse y cargarse fuera de los supuestos específicos de Bun.

## 10. SDK y protocolos

No se identifica una reescritura completa de SDK/protocol equivalente a la V2 posterior.

El trabajo de `opencode-2-0` está un nivel inferior:

- HTTP implementation;
- WebSocket adapter;
- server lifecycle;
- OAuth callback HTTP;
- process/runtime compatibility.

La posterior V2 formaliza `Protocol` y clientes generados como boundaries de producto; esta branch prepara el terreno de portabilidad necesario.

## 11. Backend

Ésta es una de las áreas centrales de la branch.

Cambios principales:

- servidor principal deja de depender directamente de `Bun.serve`;
- workspace server devuelve un objeto lifecycle en vez de un server Bun implícito;
- comandos CLI deben `await Server.listen()`;
- WebSocket/PTY necesita adapter explícito;
- Node build debe incluir migrations.

### Boundary revelado

`Server.listen` pasa de ser una thin call a Bun a una interfaz de runtime que debe estabilizar:

```text
listen
hostname
port
url
stop
websocket upgrade
```

Éste es un boundary arquitectónico que posteriormente permite otras generaciones de server refactoring.

## 12. UI

No hay evidencia de una reescritura UI general. Los cambios TUI observados son de compatibilidad:

- filesystem API consistente;
- plugin loading/path handling Windows;
- adaptación al nuevo server lifecycle.

Por tanto, UI no es el propósito primario de `opencode-2-0`.

## 13. Persistence

La persistencia cambia principalmente por deployment/runtime:

- migraciones se empaquetan para builds Node;
- se extraen abstracciones SQLite en PRs separados, según #18335.

No es todavía el rediseño event-oriented de sesión que caracteriza V2.

## 14. Qué llegó a `dev`

### Confirmado por merges

| Pieza originada/extractada de la línea | PR | Estado |
| --- | ---: | --- |
| npm/Arborist en lugar de BunProc | #18308 | merged |
| portable Process para shell/spawn | #18318 | merged |
| executable discovery/cache bin | #18320 | merged |
| Node entrypoint/build | #18324 | merged |
| Node HTTP OAuth callbacks | #18327 | merged |
| misc Windows/tool path fixes | #18328 | merged |
| Hono Node server/WebSocket adapter | #18335 | merged |

#18335 menciona además la extracción previa de SQLite abstraction.

### Resultado

La arquitectura `dev` posterior ya contiene una cantidad sustancial de esa portabilidad, aunque posteriormente vuelva a evolucionar hacia otros listeners/adapters.

## 15. Qué fue descartado

### Descartado: integrar `opencode-2-0` como mega-branch

El PR #16918 fue cerrado sin merge.

### No descartado: el objetivo Node/portable

Sus piezas fueron divididas y fusionadas individualmente.

Ésta es una distinción importante para arqueología de software:

```text
branch rejected != architecture rejected
```

En este caso, lo rechazado fue principalmente la unidad de integración/riesgo del cambio, no la dirección técnica.

## 16. Relación con `2.0` y `beta/v2`

```text
opencode-2-0 (marzo)
  |
  |  portabilidad infra
  |  Node / server / process / npm / OAuth
  +-------> piezas merged a dev
                 |
                 v
             dev evoluciona
                 |
          2.0 exploration (abril)
          session domain spike
                 |
          muchos refactors posteriores
                 |
                 v
             beta == v2 (agosto)
             arquitectura completa separada
```

No hay evidencia suficiente para afirmar que `beta/v2` sea una descendencia lineal directa de `opencode-2-0`. Sí hay evidencia de que varios prerequisites de portabilidad fueron absorbidos por `dev` antes de la V2 madura.

## Hipótesis

1. **INFERENCIA:** `opencode-2-0` fue un integration spike destinado a demostrar que OpenCode podía dejar de ser Bun-only antes de repartir la migración en cambios revisables.
2. **INFERENCIA:** el tamaño y heterogeneidad del PR #16918 probablemente motivaron su descomposición, pero no se encontró una declaración formal que atribuya el cierre específicamente al tamaño.
3. **INFERENCIA:** esta línea de trabajo ayudó a revelar boundaries que posteriormente se convirtieron en packages/services explícitos: Process, filesystem/runtime adapters, server listener, package installation y PTY transport.

## Conclusión

`opencode-2-0` es especialmente valiosa como evidencia arquitectónica porque muestra dónde estaban realmente incrustadas las dependencias de Bun. Su destino también es revelador: la branch completa fue abandonada como unidad, pero la mayoría de sus objetivos estructurales importantes fueron extraídos y fusionados en `dev`. En términos evolutivos, es un precursor infrastructural de la arquitectura multi-runtime y modular que V2 lleva más lejos.
