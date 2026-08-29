# Boundaries de dominio: Project, Session, Provider, Plugin, Config y Workspace

## Project

### Boundary observado

En `dev`, `packages/opencode/src/project/project.ts` define un `Project.Service` de aplicación que depende explícitamente de:

- `FSUtil.node`
- `AppProcess.node`
- `CrossSpawnSpawner.node`
- `ProjectV2.node`
- `ProjectDirectories.node`
- `EventV2Bridge.node`
- `RuntimeFlags.node`
- `Database.node`

El servicio posee persistencia/proyección de `ProjectTable`, actualiza sesiones/workspaces durante migraciones de identidad y usa `ProjectV2` para resolver el checkout.

**Hecho observado — confianza alta.** `ProjectV2` de core y `Project` de aplicación coexisten; el segundo depende del primero.

**Inferencia arquitectónica — confianza alta.** Project es un ejemplo claro de strangler architecture. `ProjectV2` contiene la semántica portable de resolución/identidad; el servicio legacy/app conserva compatibilidad, persistencia y emisión de eventos de producto.

`core-project-relocation` refuerza este ownership: una mudanza física del checkout se coordina desde Project y actualiza Project, Session y context epochs transaccionalmente.

## Session

### Grafo reconstruido desde `layer-node-sprompt`

```text
SessionV2
 ├─ SessionExecution
 ├─ SessionStore ── Database
 ├─ SessionProjector
 ├─ EventV2
 ├─ Database
 └─ ProjectV2
```

**Hecho observado — confianza alta.** `SessionExecution` puede sustituirse independientemente por un layer mock/noop; `SessionStore` es un puerto de persistencia sobre DB.

**Inferencia arquitectónica — confianza alta.** Session no es un objeto monolítico. El refactor separa al menos:

- API/orquestación de sesión;
- store/persistencia;
- projector/event projection;
- execution strategy;
- proyecto/contexto;
- transporte/handle operacional.

`effect/session-transport-service` añade otra frontera: el transporte por ejecución no debe poseer un Layer fresco propio; se crea como handle scoped desde un service factory.

## Config

En `packages/opencode/src/config/config.ts`, el node actual depende de:

```text
Config
 ├─ FSUtil
 ├─ Auth
 ├─ Account
 ├─ Env
 ├─ Npm
 └─ HttpClient
```

El servicio conserva estado por instancia mediante `InstanceState`, además de configuración global, merge/parsing, resolución de plugins y espera de dependencias.

**Hecho observado — confianza alta.** Config es un servicio con acceso a filesystem, auth, account, env, paquetes y red; no es un simple lector de JSON.

**Inferencia arquitectónica — confianza alta.** Config es una boundary service de producto: combina múltiples fuentes y normaliza el modelo de configuración para otros subsistemas. La línea `config-effect-v2` intenta mover partes genéricas del environment/config hacia `Effect.Config`, reduciendo el acoplamiento a este servicio legacy.

## Plugin

En `packages/opencode/src/plugin/index.ts`, el node depende de:

```text
Plugin
 ├─ EventV2Bridge
 ├─ Config
 └─ RuntimeFlags
```

El estado de hooks se guarda en `InstanceState`; se filtran eventos por directory y los hooks reciben finalizers de dispose.

**Hecho observado — confianza alta.** Plugin es un servicio global con estado contextual por instancia. También registra adapters de Workspace y carga plugins internos/externos.

**Inferencia arquitectónica — confianza alta.** El boundary interno de Plugin es lifecycle + hook dispatch; el boundary externo es el contrato del paquete `@opencode-ai/plugin`. Las ramas `effect-plugin-adapter` y `plugin-effect-runtime` existen precisamente para impedir que ambas superficies queden acopladas a la misma instancia de Effect.

## Provider

`packages/opencode/src/provider/provider.ts` sigue siendo un servicio de aplicación grande. Importa/coordina Config, Npm, Plugin, ModelsDev, Auth, Env, FSUtil, ProviderV2/ModelV2, RuntimeFlags y transforms específicos de proveedores/AI SDK. También usa `InstanceState`.

**Hecho observado — confianza alta.** Provider contiene tanto resolución de catálogo/modelos como adapters de SDK y transformaciones específicas (OpenAI, Anthropic, Vertex, Bedrock, Cloudflare, Snowflake, etc.).

**Inferencia arquitectónica — confianza alta.** Provider es uno de los boundaries todavía parcialmente monolíticos. Las ramas `effectify-provider`, `native-provider-*` y la existencia de `ProviderV2`/`ModelV2` en core indican un intento de separar:

1. identidad/catálogo de provider-model;
2. auth/configuración;
3. construcción del SDK;
4. protocol/provider transformations;
5. estado contextual.

La modularización está menos completa aquí que en filesystem/location o SessionV2.

## Workspace

`kit/effect-workspace` muestra que Workspace permanecía en el backlog cuando otros servicios ya se consideraban migrados. `kit/effect-workspace-adapters` separa el adapter Promise público del `InternalWorkspaceAdapter` Effect y registra adapters por `ProjectID`.

**Hecho observado — confianza alta.** Worktree es un adapter built-in del modelo Workspace; plugins pueden registrar adapters custom y el bridge convierte Promise a Effect.

**Inferencia arquitectónica — confianza alta.** Workspace se modela como control-plane/orchestration sobre estrategias de materialización (worktree u otras), no como parte intrínseca de Project. Project identifica el repositorio/checkout; Workspace controla entornos derivados y extensibles.

## Filesystem y Project

`FileSystem.node(location)` del core depende de Location, mientras Project posee identidad persistida de checkout.

**Inferencia arquitectónica — confianza alta.** Son boundaries complementarios:

- Project responde “qué checkout/proyecto es este y cómo persiste su identidad”;
- Location responde “qué contexto activo representa este ref”;
- FileSystem responde “cómo operar/buscar dentro de ese contexto”.

## Dependency graph simplificado de la shell legacy

```text
                     AppRuntime
                        │
       ┌────────────────┼─────────────────┐
       │                │                 │
     Config           Project           Plugin
       │                │                 │
       ├─Auth           ├─ProjectV2       ├─Config
       ├─Account        ├─DB              ├─Events
       ├─Npm            ├─Events          └─Flags
       └─FS/HTTP        └─FS/Process
          │                │
          └──────┬─────────┘
                 ▼
              Provider
       Config/Auth/Plugin/Models
                 │
                 ▼
              Session/LLM
```

No representa todos los edges, sino los boundaries dominantes observados.

## Core location graph

En paralelo, `packages/core/src/location-services.ts` reúne una generación más desacoplada:

```text
Location
├─ Config v2 / Policy / Permission
├─ FileSystem / Watcher / Pty
├─ Agent / Skill / Command
├─ Plugin v2 / Integration / Catalog
├─ ToolRegistry / BuiltInTools
├─ SystemContext
└─ SessionRunner / Model / LLM / Snapshot
```

**Inferencia arquitectónica — confianza alta.** La coexistencia de ambos graphs explica muchas duplicidades nominales (`Project`/`ProjectV2`, legacy Config/core Config, Plugin/plugin v2): son líneas de migración, no necesariamente duplicación accidental sin propósito.

## Boundaries más maduros vs más transicionales

| Subsistema | Estado inferido |
|---|---|
| LayerNode / DI | integrado y canónico |
| Location / filesystem core | integrado, boundary fuerte |
| Project | híbrido legacy + core |
| Session | separación avanzada en core, compatibilidad en app |
| Config | Effect service integrado; extracción genérica parcial |
| Plugin | lifecycle integrado; boundary público/Effect activamente desacoplado |
| Provider | todavía amplio/heterogéneo; extracción parcial |
| Workspace | históricamente tardío; adapters muestran separación en progreso |

Esta clasificación es una inferencia basada en las ramas y en `dev`, no una etiqueta oficial del proyecto.