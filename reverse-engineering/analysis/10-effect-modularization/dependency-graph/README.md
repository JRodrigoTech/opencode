# `LayerNode`: grafo de dependencias y dependency injection

## Qué problema resuelve

**Hecho observado — confianza alta.** Antes de la familia `layer-node-*`, muchos tests y aggregate layers construían manualmente árboles de `Layer.provide`, `Layer.provideMerge` y `Layer.mergeAll`. Esto obligaba al caller a conocer dependencias transitivas y hacía fácil crear instancias duplicadas o wiring incoherente.

Las ramas `layer-node-cproject`, `layer-node-integrate`, `layer-node-location`, `layer-node-locfs` y `layer-node-sprompt` sustituyen ese wiring por nodes declarativos.

## Modelo vigente en `dev`

`packages/core/src/effect/layer-node.ts` define tres clases lógicas de node:

- `layer`: componente con implementación;
- `unbound`: dependencia/puerto cuya implementación debe venir de fuera;
- `group`: composición de varios nodes.

Cada node puede declarar:

- nombre;
- `service` producido;
- `layer`/implementación;
- dependencias directas;
- tag de lifetime/arquitectura.

El sistema también contiene checks de tipos para detectar servicios requeridos por un Layer que no aparezcan entre sus deps.

**Inferencia arquitectónica — confianza alta.** `LayerNode` convierte el grafo arquitectónico en datos ejecutables y comprobables. La lista `deps` no es documentación auxiliar: participa en compilación, replacements y validación.

## Tags: dirección de dependencias

En `packages/core/src/effect/app-node.ts`:

```ts
tags = LayerNode.tags({
  location: ["global"],
  global: [],
})
```

**Hecho observado — confianza alta.** Un node `location` puede consumir nodes `global`; un node `global` no puede depender de `location`.

**Inferencia arquitectónica — confianza alta.** Esto es una regla de arquitectura hexagonal/layered codificada en tipos: el contexto más largo/estable no puede capturar accidentalmente un servicio contextual más corto. La inversión se resuelve mediante factories/maps (`LocationServiceMap`) en vez de dependencia directa global→location.

## Replacements: seam de testing y host integration

### Project

`layer-node-cproject` reemplaza una construcción manual de DB + directories + Git + FS por:

```ts
LayerNode.buildLayer(ProjectV2.node, {
  replacements: [
    LayerNode.replace(Database.node, Database.layerFromPath(":memory:"))
  ]
})
```

### Integration

`layer-node-integrate` declara:

```text
Integration
 ├─ Credential
 │   └─ Database
 └─ EventV2
```

El test sustituye `Credential.node` por un mock sin reescribir Integration.

### Session

`layer-node-sprompt` hace explícito:

```text
SessionV2
 ├─ SessionExecution
 ├─ SessionStore ── Database
 ├─ SessionProjector
 ├─ EventV2
 ├─ Database
 └─ ProjectV2
```

El test reemplaza `Database` y `SessionExecution`.

**Inferencia arquitectónica — confianza alta.** El replacement seam define qué componentes se consideran sustituibles. Database, execution strategy, credential store y Location aparecen como puertos naturales.

## Filesystem como función de Location

`layer-node-locfs` declara:

```text
FileSystem(location)
 ├─ FSUtil
 ├─ FileSystemSearch(location)
 │   ├─ FSUtil
 │   ├─ Ripgrep
 │   └─ Location
 └─ Location
```

El test fabrica un node `Location.Service` para un directorio temporal y compila `FileSystem.node(activeLocation)`.

**Hecho observado — confianza alta.** La abstracción de filesystem de core nuevo no es simplemente “filesystem del proceso”: está parametrizada por el contexto location.

## Hoisting y `LocationServiceMap`

En `dev`, `packages/core/src/location-services.ts` agrupa decenas de nodes location-scoped. Al construir un `Location.Ref`:

1. se reemplaza `Location.node` por `Location.boundNode(ref)`;
2. `LayerNode.hoist(..., globalTag, replacements)` extrae dependencias globales;
3. se compila un graph location fresco;
4. se provee el subgrafo global hoisted;
5. el resultado se almacena en un `LayerMap` con TTL.

El comentario del código indica que los replacements deben aplicarse durante el hoist porque pueden introducir nuevas dependencias tagged.

**Inferencia arquitectónica — confianza alta.** `hoist` es el mecanismo que evita duplicar servicios globales dentro de cada location y, a la vez, permite que cada location tenga estado/recursos propios.

## `unbound` como puerto

Ejemplos actuales:

- `Location.node` es `unbound` hasta ligarse a un `Location.Ref`.
- `LocationServiceMap.node` es `unbound` global y `AppNodeBuilder` sintetiza su implementación si el graph la necesita.
- `InstanceStore.bootstrapNode` expone solo el tag `InstanceBootstrap.Service`; `AppNodeBuilderV1` lo reemplaza por `InstanceBootstrap.node` completo.

**Inferencia arquitectónica — confianza alta.** `unbound` representa dependency inversion real: una capa declara que necesita un puerto, y el host/composition root decide la implementación sin forzar import del grafo concreto.

## Cycle detection y compilación

La implementación actual recorre el graph, detecta ciclos y compila recursivamente las dependencias. Replacements conservan constraints de output/error/tag.

**Inferencia arquitectónica — confianza alta.** OpenCode ha pasado de DI “por convención” a un pequeño sistema de componentes tipado. `LayerNode` funciona como una descripción arquitectónica intermedia entre Effect `Layer` puro y el composition root de producto.

## Grafo conceptual de alto nivel

```text
GLOBAL
├─ Database
├─ EventV2
├─ Project / ProjectV2
├─ Auth / Account / Npm / ModelsDev
├─ App services legacy
├─ InstanceStore
└─ LocationServiceMap (factory/cache de subgrafos)
       │
       ▼
LOCATION(ref)
├─ Location
├─ Config (core v2)
├─ Policy / Permission
├─ FileSystem / Watcher / Pty
├─ Agent / Commands / Skills
├─ Plugin v2 / Integration
├─ ToolRegistry / BuiltInTools
├─ SystemContext
└─ Session runner / LLM / Snapshot
```

El detalle exacto evoluciona, pero el boundary global→location está explícitamente codificado en `dev`.