# 10 — Effect, modularización y boundaries arquitectónicos

## Objetivo

Este análisis usa las ramas de refactor de Effect, namespaces, facades, `LayerNode` y extracción de `core` como evidencia para reconstruir los **boundaries reales** de OpenCode. El foco no es describir archivos, sino identificar qué piezas se intentaron convertir en componentes independientes, cómo se compone su grafo de dependencias, quién posee el estado y qué líneas de refactor sobrevivieron en `dev`.

Baseline analizada: `dev` (`dc4449df0d52199704ea4989a5a993ebbc605612`).

## Conclusión principal

**Hecho observado — confianza alta.** La arquitectura vigente contiene un grafo de dependencias ejecutable construido con `LayerNode`; `packages/opencode/src/effect/app-runtime.ts` compone `AppLayer` desde nodes de servicio y `packages/core/src/effect/app-node.ts` impone dos tags arquitectónicos: `global` y `location`.

**Inferencia arquitectónica — confianza alta.** Las ramas estudiadas muestran una transición desde un monolito de módulos con funciones async y estado implícito hacia una arquitectura de **servicios Effect con puertos explícitos, composición declarativa, lifetimes diferenciados y adapters en los bordes**. La unidad de modularidad relevante no es el fichero ni el `namespace`: es el servicio + su node + sus dependencias + el scope que posee su estado.

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

Existe además un lifetime operacional: `effect/session-transport-service` reemplaza un `Layer.fresh` por operación por un servicio fábrica cuyo `make(input)` devuelve un handle scoped.

## Boundaries confirmados

| Boundary | Evidencia | Lectura arquitectónica |
|---|---|---|
| Global ↔ Location | `packages/core/src/effect/app-node.ts` | `location` puede depender de `global`; la dirección inversa queda prohibida por tags. |
| App orchestration ↔ Core reusable | `packages/opencode/...`, `packages/core/...`, `packages/core/AGENTS.md` | Core se diseña Node-compatible/reusable; app conserva CLI/server/compatibilidad. |
| Project legacy ↔ ProjectV2 | `packages/opencode/src/project/project.ts` | Patrón strangler: el servicio app/persistencia delega semántica al core nuevo. |
| Instance lifecycle ↔ service lifetime | `InstanceStore`, `InstanceState`, `InstanceBootstrap` | Un service global puede poseer estado aislado por directory con cleanup explícito. |
| Location lifecycle ↔ global services | `LocationServiceMap`, `location-services.ts` | Core materializa un subgrafo por `Location.Ref` y hoistea deps globales. |
| Plugin authoring ↔ host runtime | `effect-plugin-adapter`, `plugin-effect-runtime` | Effect queda aislado del contrato portable mediante Promise/Standard Schema adapters. |
| Service API ↔ Promise facade | ramas `facade/*` y eliminación de `makeRuntime` | Las facades internas fueron scaffolding migratorio; adapters externos pueden seguir siendo válidos. |
| Filesystem platform ↔ filesystem contextual | FSUtil/App filesystem y `FileSystem.node(Location)` | El filesystem de dominio se interpreta respecto a Location. |
| Environment ↔ service graph | `config-effect-v2` | Configuración migra de `process.env` import-time a `Effect.Config`/`ConfigProvider`. |

## Organización por familia arquitectónica

- [`branches/README.md`](./branches/README.md) — inventario, cobertura y cronología de branches.
- [`effect-services/README.md`](./effect-services/README.md) — Effect Services, facades y runtimes.
- [`dependency-graph/README.md`](./dependency-graph/README.md) — `LayerNode`, DI, tags, replacements, hoisting y compilation.
- [`lifetimes-filesystem/README.md`](./lifetimes-filesystem/README.md) — ownership de estado, scopes, instance/location/operation y filesystem.
- [`core-extraction/README.md`](./core-extraction/README.md) — extracción de `core`, Node compatibility, ports y adapters.
- [`domains/README.md`](./domains/README.md) — Project, Session, Config, Plugin, Provider y Workspace.
- [`lineage/README.md`](./lineage/README.md) — refactors integrados, parciales, reemplazados o reinterpretados.

## Hallazgos clave

1. `LayerNode` terminó convertido en arquitectura ejecutable, no solo helper de tests.
2. La regla de dependencia más fuerte es `location → global`; `LocationServiceMap` permite cruzar ese boundary sin invertirlo.
3. `InstanceState` desacopla lifetime del service y lifetime del estado; los subgrafos Location hacen explícito el mismo problema a escala mayor.
4. Project y Session se descomponen alrededor de identidad, persistencia, proyección, ejecución y contexto.
5. Plugins son un boundary de runtime real: no se presupone identidad compartida de Effect/Schema entre host y extensión.
6. `packages/core` representa portabilidad de dominio/runtime, no una mera colección de utilidades.
7. Provider y Workspace siguen mostrando más señales de transición y concentración de responsabilidades que LayerNode, SessionV2 o filesystem location-scoped.
8. Los refactors no integrados siguen siendo evidencia cuando sus seams reaparecen en `dev` bajo una implementación posterior.

## Convención de evidencia

- **Hecho observado**: demostrable en código, diff, branch o commit inspeccionado.
- **Inferencia arquitectónica**: conclusión derivada de uno o varios hechos.
- **Confianza alta/media/baja**: fuerza de la evidencia disponible.

El nombre de una branch no se toma por sí solo como prueba de intención. En familias con muchas ramas solapadas se combina inventario por patrón con análisis profundo de commits representativos.