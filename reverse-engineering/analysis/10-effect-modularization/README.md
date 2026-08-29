# 10 — Effect, modularización y boundaries arquitectónicos

## Objetivo

Este análisis usa las ramas de refactor de Effect, namespaces, facades, `LayerNode` y extracción de `core` como evidencia para reconstruir los **boundaries reales** de OpenCode. El foco no es describir el estilo de código, sino identificar qué piezas se intentaron convertir en servicios independientes, cómo se compone su grafo de dependencias, quién posee el estado y qué líneas de refactor sobrevivieron en `dev`.

Baseline: `dev` (`dc4449df0d52199704ea4989a5a993ebbc605612` en el momento del análisis).

## Conclusión principal

**Hecho observado — confianza alta.** La arquitectura vigente ya no depende únicamente de convenciones informales. `dev` contiene un grafo de dependencias ejecutable construido con `LayerNode`; `packages/opencode/src/effect/app-runtime.ts` compone `AppLayer` desde nodos de servicio y `packages/core/src/effect/app-node.ts` impone dos tags de lifetime/dependencia: `global` y `location`.

**Inferencia arquitectónica — confianza alta.** Las ramas estudiadas muestran una transición desde un monolito de módulos con funciones async y estado implícito hacia una arquitectura de **servicios Effect con puertos explícitos, composición declarativa, lifetimes diferenciados y adapters en los bordes**. La unidad de modularidad relevante no es el fichero ni el `namespace`: es el servicio + su nodo + sus dependencias + el scope que posee su estado.

## Arquitectura reconstruida

```text
                         ┌─────────────────────────────────┐
                         │        AppRuntime / global       │
                         │ ManagedRuntime + shared memoMap  │
                         └───────────────┬─────────────────┘
                                         │
                 ┌───────────────────────┼────────────────────────┐
                 │                       │                        │
        ┌────────▼────────┐     ┌────────▼─────────┐     ┌────────▼─────────┐
        │ Legacy/app svcs │     │ Core global svcs │     │ InstanceStore     │
        │ Config, Plugin, │     │ DB, ProjectV2,   │     │ directory→context │
        │ Provider, etc.  │     │ EventV2, etc.    │     └────────┬─────────┘
        └────────┬────────┘     └────────┬─────────┘              │
                 │                       │                 InstanceState
                 │                       │                ScopedCache(dir)
                 │                       │
                 │              LocationServiceMap
                 │                 LayerMap(ref)
                 │                       │
                 │               ┌───────▼────────┐
                 │               │ location graph │
                 │               │ tools, agent,  │
                 │               │ plugin v2, FS, │
                 │               │ policy, LLM... │
                 │               └────────────────┘
                 │
                 └──── adapters/bridges durante la migración
```

Además aparece un cuarto lifetime para recursos operacionales: **handles por operación**. `effect/session-transport-service` reemplaza un `Layer.fresh` por sesión por un servicio fábrica estático cuyo `make(input)` devuelve un handle propietario de `Scope`/`AbortController`.

## Boundaries confirmados

| Boundary | Evidencia en `dev` | Lectura arquitectónica |
|---|---|---|
| Global ↔ Location | `packages/core/src/effect/app-node.ts` | `location` puede depender de `global`; el sentido inverso queda prohibido por tags. |
| App orchestration ↔ Core reusable | `packages/opencode/...` frente a `packages/core/...`; `packages/core/AGENTS.md` | `core` se diseña como runtime Node-compatible reutilizable por desktop/SDK; app conserva CLI/server y compatibilidad. |
| Project legacy ↔ ProjectV2 | `packages/opencode/src/project/project.ts` depende de `ProjectV2.node` | Patrón strangler: servicio app/persistencia envuelve/resuelve mediante core nuevo. |
| Instance lifecycle ↔ service lifetime | `InstanceStore`, `InstanceState`, `InstanceBootstrap` | Servicios pueden ser globales mientras el estado mutable se cachea por directory y se dispone por instancia. |
| Location lifecycle ↔ global services | `LocationServiceMap`, `location-services.ts` | Core nuevo materializa un subgrafo fresco por `Location.Ref` y hoistea dependencias globales. |
| Plugin authoring ↔ host runtime | `effect-plugin-adapter`, `plugin-effect-runtime` | El contrato público tiende a ser runtime-neutral; Effect queda como authoring/runtime interno y se adapta en el borde. |
| Service API ↔ Promise facade | `effect/define-service-helper` vs `effect/kill-tuiconfig-installation-facades` | Las facades Promise fueron una etapa transitoria; la dirección final favorece `Service` + `AppRuntime`. |
| Filesystem global ↔ filesystem contextual | `FSUtil`/`AppFileSystem` y `core FileSystem.node(Location)` | El nuevo core hace explícito que búsqueda/mutación de archivos depende de una location, no solo del proceso. |

## Documentos

1. [`01-branch-inventory.md`](./01-branch-inventory.md) — inventario de familias y cobertura.
2. [`02-effect-services-and-facades.md`](./02-effect-services-and-facades.md) — migración a servicios, runtimes y facades.
3. [`03-layer-node-dependency-graph.md`](./03-layer-node-dependency-graph.md) — DI, tags, replacements, hoisting y graph compilation.
4. [`04-state-lifetimes-and-filesystem.md`](./04-state-lifetimes-and-filesystem.md) — ownership de estado, scopes, instances, locations y filesystem.
5. [`05-core-extraction.md`](./05-core-extraction.md) — extracción de `core`, compatibilidad Node y dependency inversion.
6. [`06-domain-boundaries.md`](./06-domain-boundaries.md) — Project, Session, Provider, Plugin, Config y Workspace.
7. [`07-refactor-lineage.md`](./07-refactor-lineage.md) — qué fue integrado, parcialmente integrado, reemplazado o abandonado.

## Convención de evidencia

Cada documento usa explícitamente:

- **Hecho observado**: demostrable en código, diff, branch o commit inspeccionado.
- **Inferencia arquitectónica**: conclusión derivada de varios hechos.
- **Confianza alta/media/baja**: fuerza de la evidencia disponible.

No se considera que el nombre de una branch pruebe por sí solo una intención. Cuando una rama solo aporta señal nominal o periférica se indica como tal.
