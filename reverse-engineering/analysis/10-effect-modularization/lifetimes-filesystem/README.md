# Ownership de estado, lifetimes y filesystem

## Tesis

**Inferencia arquitectónica — confianza alta.** Los refactors revelan que OpenCode tiene varios lifetimes distintos y que gran parte de la modularización consistió en dejar de confundirlos:

1. **process/global** — servicios compartidos por el runtime;
2. **location** — servicios ligados a un checkout/workspace location;
3. **instance/directory** — estado legacy por instancia de aplicación;
4. **operation/session transport handle** — recursos efímeros adquiridos y liberados por operación.

## Global

`AppRuntime` crea un `ManagedRuntime` a partir de `AppLayer` y usa un `memoMap` compartido.

**Hecho observado — confianza alta.** Servicios como DB, auth, modelos, composición de proyecto y factories de contextos se materializan dentro de este runtime o son hoisted hacia él.

**Inferencia arquitectónica.** “Global” no significa necesariamente singleton mutable sin control; significa que su Layer tiene lifetime del composition root y que puede poseer caches/scopes gestionados por Effect.

## InstanceStore: identidad y lifecycle por directory

En `packages/opencode/src/project/instance-store.ts`, `InstanceStore.Service` posee un `Map<string, Entry>` cuyo key es el path resuelto del directorio. Cada `Entry` contiene un `Deferred<InstanceContext>`.

Flujo observado:

```text
load(directory)
  ├─ normaliza path
  ├─ reutiliza Deferred existente si hay load concurrente
  ├─ Project.fromDirectory(directory)
  ├─ crea InstanceContext
  └─ ejecuta InstanceBootstrap con InstanceRef
```

`reload` sustituye la entrada y dispone el contexto anterior; `dispose*` ejecuta los disposers registrados y emite `server.instance.disposed`; el Scope del servicio añade un finalizer que dispone todas las instancias.

**Inferencia arquitectónica — confianza alta.** `InstanceStore` es el propietario de la identidad/lifecycle de la instancia legacy. Centraliza deduplicación de boot, reload y cleanup, evitando que cada servicio invente su propio lifecycle.

## InstanceState: estado por instancia dentro de servicios globales

`packages/opencode/src/effect/instance-state.ts` implementa `InstanceState.make` con `ScopedCache<string, A>` indexado por el directory de `InstanceRef`. Registra un disposer que invalida esa key cuando se elimina una instancia.

**Hecho observado — confianza alta.** Config, Plugin, Provider y otros servicios migrados usan `InstanceState` para mantener estado contextual sin recrear el Service completo por cada instancia.

**Inferencia arquitectónica — confianza alta.** Se separan dos cosas que antes podían estar acopladas:

```text
Service object lifetime: global
State lifetime:          instance/directory
```

Esto reduce el número de layers dinámicos y permite compartir infraestructura estable mientras se conserva aislamiento de estado.

## InstanceBootstrap: ordering como boundary

`packages/opencode/src/project/bootstrap.ts` construye un servicio que, al ejecutar `run`:

1. obtiene el contexto de instancia;
2. fuerza `Config.get()` para cargar configuración;
3. inicializa Plugin antes que el resto porque puede mutar config;
4. materializa en paralelo LSP, ShareNext, Format, VCS, Snapshot y Project.

**Hecho observado — confianza alta.** Su node declara esas dependencias explícitamente.

**Inferencia arquitectónica.** Bootstrap es un boundary de orchestration, no dominio. Su responsabilidad es ordering/materialization, mientras los servicios individuales conservan ownership de su trabajo y sus scopes.

## Location: generación más explícita del contexto

En `packages/core/src/location-services.ts`, `LayerMap` crea un subgrafo para cada `Location.Ref`, aplica un `Location.boundNode(ref)`, compila nodes location-scoped con `Layer.fresh` y usa TTL de 60 minutos.

**Hecho observado — confianza alta.** El graph contiene FileSystem, Agent, Config, Policy, Plugin v2, Integration, ToolRegistry, SystemContext, SessionRunner, Snapshot y otros servicios.

**Inferencia arquitectónica — confianza alta.** `Location` es una evolución más declarativa que el mecanismo legacy de InstanceState: el conjunto completo de servicios location-aware puede ser materializado como subgrafo aislado, mientras las deps globales se hoistean.

No deben confundirse ambos mecanismos: coexisten durante una migración arquitectónica.

## Handles por operación

`effect/session-transport-service` deja de usar un layer fresco por sesión/operación y crea un servicio fábrica con un `make(input)` que devuelve un handle propietario de los recursos operacionales.

**Inferencia arquitectónica — confianza alta.** Cuando la identidad de un recurso es “una ejecución” y no “un componente”, un handle scoped es más correcto que un Layer. La modularización también consistió en elegir la abstracción de lifetime apropiada.

## Filesystem: varias abstracciones, distintos niveles

### FSUtil / App filesystem

La línea Effect introduce/adapta APIs de filesystem como servicios compartidos, útiles para operaciones generales y para desacoplar Node/Bun APIs.

### Core `FileSystem` location-scoped

`layer-node-locfs` hace que `FileSystem.node(location)` dependa de:

- `FSUtil.node`;
- `FileSystemSearch.node(location)`;
- el propio `Location`.

`FileSystemSearch` depende a su vez de FSUtil, Ripgrep y Location.

**Hecho observado — confianza alta.** El test puede inyectar una Location temporal y compilar solo el filesystem necesario.

**Inferencia arquitectónica — confianza alta.** Hay un boundary claro entre:

- **capacidad de filesystem de plataforma** — APIs para leer/escribir/probar paths;
- **filesystem de dominio/location** — búsqueda y operaciones interpretadas respecto a un checkout/contexto.

## Project y paths persistidos

`core-project-relocation` muestra que mover un checkout obliga a actualizar de forma transaccional ProjectTable, SessionTable, ProjectDirectoryTable y context epochs.

**Inferencia arquitectónica — confianza alta.** El path no es solo un argumento de filesystem; forma parte de la identidad persistida Project/Session. Por ello, la abstracción Project debe mediar cambios de checkout en vez de delegarlos a un utility de FS.

## Resumen de ownership

| Recurso/estado | Owner observado | Lifetime |
|---|---|---|
| composición app / servicios base | `AppRuntime` / Scope | process/global |
| DB y servicios core globales | graph global | process/global |
| mapa de contexts location | `LocationServiceMap` | global factory/cache |
| servicios tools/agent/fs por location | subgrafo `Location.Ref` | location + TTL/scope |
| context legacy por directory | `InstanceStore` | instance |
| estado interno de Config/Plugin/etc. | `InstanceState` | instance, dentro de service global |
| transporte/recursos de ejecución | handle `make(...)` | operación |

**Conclusión:** una de las claves del diseño moderno de OpenCode es que *dependency lifetime* y *state lifetime* ya no tienen por qué coincidir.