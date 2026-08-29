# Extracción de `core` y modularización cross-runtime

## `packages/core` como boundary

**Hecho observado — confianza alta.** `core-node-guidance` (`3ab166579f2b…`) modifica `packages/core/AGENTS.md` para exigir que Core ejecute en Node.js y prohíbe globals/imports Bun-only en shared runtime code; los adapters específicos de Bun deben tener equivalentes Node.

**Inferencia arquitectónica — confianza alta.** Esto define un boundary de plataforma explícito: `packages/core` está pensado para consumidores que no son el binario Bun principal (por ejemplo desktop/SDK/otros hosts), mientras `packages/opencode` puede conservar orchestration y adaptadores específicos del producto.

## Qué se extrae

La branch `core` (`a0f7731e…`) es WIP y no debe leerse como arquitectura final, pero muestra la dirección:

- runtime/memoization de Effect;
- observabilidad;
- filesystem abstractions;
- utilidades y servicios compartidos;
- progresiva eliminación de namespaces internos;
- movimiento de dominio reusable.

En `dev`, `packages/core` ya contiene piezas sustanciales de dominio/runtime: Database, ProjectV2, SessionV2, EventV2, Location, filesystem contextual, plugins v2, tool registry, agent/skill/config v2 y `LayerNode`.

**Inferencia arquitectónica — confianza alta.** La extracción no sigue “utils vs business logic”. El criterio es más parecido a **portable reusable domain/runtime vs product orchestration/legacy integration**.

## Node graph como contrato de composición de Core

`core-node-consumers` (`12395c37…`) cambia consumers que construían `defaultLayer` manualmente para usar `LayerNode.compile(node)`. También hace que nodes aggregate como cleanup declaren deps concretas.

**Hecho observado — confianza alta.** El graph canónico se vuelve fuente de verdad para construir servicios, incluso cuando se mantiene una facade Promise externa.

**Inferencia arquitectónica.** El objetivo no es obligar a todo consumidor a escribir Effect. El objetivo es que exista una única definición de dependencias debajo de todas las superficies de consumo.

## Platform adapters

El core actual separa node definitions de platform nodes (`effect/app-node-platform`) y puede hoistear/proveer capacidades como filesystem/http/process desde el host.

**Inferencia arquitectónica — confianza alta.** Esta es dependency inversion clásica:

```text
core domain/service
      │ necesita capability
      ▼
Effect service/tag / unbound node
      ▲
      │ implementa
Node adapter / Bun adapter / host composition
```

La regla Node-compatible impide que la dependencia se invierta accidentalmente hacia Bun.

## `AppNodeBuilder` y compatibilidad entre generaciones

`packages/core/src/effect/app-node-builder.ts` detecta si el graph requiere `LocationServiceMap`; si no existe replacement, construye el mapa con los replacements activos y lo inyecta como node global.

`packages/opencode/src/effect/app-node-builder-v1.ts` añade otro replacement: el `InstanceStore.bootstrapNode` unbound se satisface con `InstanceBootstrap.node`.

**Hecho observado — confianza alta.** Core no necesita importar la implementación pesada del bootstrap legacy; solo conoce el puerto/tag.

**Inferencia arquitectónica — confianza alta.** Este patrón permite que `core` y `opencode` evolucionen a ritmos diferentes. La aplicación actúa como host que conecta puertos de core/legacy, reduciendo ciclos de imports.

## `LocationServiceMap` como inversión global→contextual

Un servicio global no puede depender directamente de un node `location` debido a las reglas de tags. El sistema resuelve la necesidad de acceder a servicios contextuales mediante un servicio global que **fabrica/cachea subgrafos por Location.Ref**.

**Inferencia arquitectónica — confianza alta.** Esta es una forma de inversión de control: la capa global depende de una capability “obtener contexto location”, no de una instancia concreta de los servicios location.

## Plugin boundary y neutralidad de runtime

`effect-plugin-adapter` (`2bc9f6ea…`) declara explícitamente que los plugins authored con Effect se compilan/adaptan al contrato runtime-neutral Promise + Standard Schema.

`plugin-effect-runtime` (`b7d0582a…`) resuelve un problema más fino: distintas copias de Effect no comparten identidad de AST/schema. El adapter convierte schemas creados por el runtime autor a Standard Schema/JSON Schema antes de cruzar al host y rechaza schemas nativos extranjeros no preparados.

**Hecho observado — confianza alta.** El boundary se prueba usando una copia separada de Effect.

**Inferencia arquitectónica — confianza alta.** El plugin API es deliberadamente un **ABI lógico** separado del runtime interno. Esto reduce coupling a:

- versión/instancia concreta de Effect;
- identidad de clases/AST;
- implementación de tool schemas;
- lifecycle interno del host.

## Workspace adapters: otro ejemplo de puerto público vs modelo interno

`kit/effect-workspace-adapters` (`38a44903…`) renombra la interfaz interna a `InternalWorkspaceAdapter` y distingue explícitamente el `PluginWorkspaceAdapter` público. `fromPromiseAdapter` traduce el adapter externo a operaciones Effect internas; los adapters registrados se almacenan por `ProjectID`.

**Inferencia arquitectónica — confianza alta.** La arquitectura evita convertir una interfaz pública extensible en dependencia directa del dominio interno. El adapter layer es el anti-corruption layer entre ambos modelos.

## ConfigProvider: inversión del entorno

`config-effect-v2` (`0552458f…`) elimina parte del acceso directo a `process.env` para DB/WebSearch. `Effect.Config` se evalúa al construir el layer y tests pueden proveer un `ConfigProvider` alternativo.

**Inferencia arquitectónica — confianza alta.** Incluso el “environment” pasa a ser un puerto. Esto es importante para Core porque un host Node/desktop/test no debe depender de un snapshot global de variables leído en import-time.

## Qué permanece fuera de Core

**Hecho observado — confianza alta.** En `dev`, `packages/opencode` aún contiene servicios grandes y de producto: Config v1/compat, Plugin legacy, Provider legacy/AI SDK integration, server/CLI, Project facade/persistence integration, InstanceStore, Workspace legacy y bridges.

**Inferencia arquitectónica — confianza alta.** La extracción es incompleta deliberadamente o en progreso. `packages/opencode` funciona como **strangler shell**: mantiene APIs y persistencia históricas mientras delega cada vez más conceptos a nodes de Core.

## Boundary resultante

```text
packages/opencode
  product shell / compatibility / server / CLI
  │
  ├─ adapters + bridges
  ├─ legacy services con InstanceState
  └─ composition root
          │
          ▼
packages/core
  portable domain + Effect graph
  global/location services
  ports/unbound nodes
          │
          ▼
platform adapters / host implementations
```

**Conclusión:** la modularización de `core` es simultáneamente separación de paquetes, dependency inversion y preparación para múltiples runtimes/hosts.