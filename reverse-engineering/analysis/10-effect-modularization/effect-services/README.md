# Effect services, facades y runtimes

## Patrón de migración observado

**Hecho observado — confianza alta.** Las ramas Effect repiten una secuencia reconocible:

```text
módulo con funciones/estado global
        ↓
Context.Service + Layer
        ↓
facade Promise respaldada por makeRuntime
        ↓
callers migrados a Service.use / Effect
        ↓
facade eliminada
        ↓
servicio incorporado al AppRuntime / LayerNode graph
```

La rama `kit/effect-workspace` incluso contiene un checklist histórico que describe servicios “fully migrated” como aquellos con namespace único, `InstanceState` cuando corresponde y facade aplanada/eliminada.

## `makeRuntime`: solución transitoria

**Hecho observado — confianza alta.** Las primeras migraciones usan helpers `makeRuntime(Service, defaultLayer)` para conservar APIs como:

```ts
export async function get() {
  return runPromise((svc) => svc.get())
}
```

Esto permite migrar la implementación sin cambiar todos los callers de una vez.

`effect/kill-tuiconfig-installation-facades` (`9f290d4381e1…`) elimina explícitamente ese patrón en `TuiConfig`, `Installation` y `SessionCompaction`. Los callers pasan a ejecutar `Service.use(...)` desde `AppRuntime`; los tests obtienen hooks dedicados en lugar de espiar la facade.

**Inferencia arquitectónica — confianza alta.** `makeRuntime` solucionaba compatibilidad, pero creaba varios composition roots implícitos. Su eliminación reduce el riesgo de instanciar servicios duplicados con caches/scopes distintos y hace visible quién controla el lifetime.

## `AppRuntime` como composition root de aplicación

**Hecho observado — confianza alta.** En `dev`, `packages/opencode/src/effect/app-runtime.ts` agrupa una lista amplia de nodes: Config, Auth, Plugin, Provider, Agent, Session, MCP, ToolRegistry, Project, Workspace, Worktree, Installation, etc. `AppLayer` se construye con `AppNodeBuilderV1.build(LayerNode.group(...))`, se añade observabilidad y se crea un `ManagedRuntime` con `memoMap` compartido.

**Inferencia arquitectónica — confianza alta.** El runtime deja de pertenecer a cada servicio. La aplicación posee un composition root global, y los servicios aportan definición + dependencias, no su propio universo de ejecución.

## Tests como evidencia de dependency inversion

`facade/config` (`591e197c…`) sustituye `AppRuntime.runPromise(...)` por:

```ts
Effect.runPromise(
  Config.Service.use(...).pipe(Effect.provide(Config.defaultLayer))
)
```

**Hecho observado — confianza alta.** El test ya no necesita levantar todo el runtime de aplicación.

**Inferencia arquitectónica — confianza alta.** Un buen boundary no solo se puede llamar: también se puede materializar de forma aislada. Esta rama muestra que Config se considera un componente construible independientemente.

## Facades “delgadas” que sobreviven

No toda API Promise es necesariamente deuda. En `core-node-consumers` (`12395c37…`), `Npm` conserva funciones async externas, pero el runtime pasa de `defaultLayer` manual a `LayerNode.compile(node)`.

**Inferencia arquitectónica — confianza media-alta.** Hay dos clases distintas de facade:

1. **facade interna accidental**, que es eliminada porque oculta DI/lifetime;
2. **adapter de boundary**, que puede mantenerse para consumidores Promise/Node siempre que su implementación compile el mismo node graph canónico.

Esta diferencia es importante para no interpretar “eliminar todas las facades” como objetivo universal.

## SessionTransport: cuándo un Layer no debe representar una operación

**Hecho observado — confianza alta.** `effect/session-transport-service` transforma el transporte de sesión en un servicio fábrica estático. En lugar de construir un `Layer.fresh` por sesión/operación, el servicio expone `make(input)` y devuelve un handle que posee recursos como scope/abort controller.

**Inferencia arquitectónica — confianza alta.** Un `Layer` debe representar infraestructura/composición relativamente estable; el estado efímero de una operación pertenece a un handle adquirido dentro de un scope. Este refactor separa claramente:

```text
Service lifetime       Operation lifetime
---------------        ------------------
transport factory  ->  session transport handle
static DI              AbortController / Scope / cleanup
```

## Namespaces y servicios

`kit/ns-project` y `kit/ns-session` eliminan namespaces pero conservan `Service`, `Interface` y `layer`.

**Hecho observado — confianza alta.** El cambio no altera necesariamente el grafo de dependencias; altera la superficie de módulo.

**Inferencia arquitectónica — confianza alta.** En OpenCode moderno hay que diferenciar:

- **namespace/barrel**: API y ergonomía de imports;
- **Service**: puerto de comportamiento;
- **Layer**: implementación/construcción;
- **LayerNode**: metadata del componente y sus dependencias;
- **Runtime/Scope**: ownership del lifetime.

Confundir estas capas lleva a una lectura errónea del diseño.

## Dirección final detectada

**Hecho observado — confianza alta.** `dev` conserva `AppRuntime`, `LayerNode`, servicios Effect y adapters específicos; muchas facades locales y wrappers redundantes de `Effect.gen` desaparecieron.

**Inferencia arquitectónica — confianza alta.** La dirección consolidada es: **un runtime de aplicación compartido, subgrafos scoped cuando el dominio lo exige, y adapters explícitos en los bordes Promise/plugin/CLI**, no un runtime privado por módulo.