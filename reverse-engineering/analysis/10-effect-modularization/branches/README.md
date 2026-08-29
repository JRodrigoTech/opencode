# Inventario de branches y familias

## Alcance

Se localizaron por nombre las familias `effect/*`, `effect-*`, `kit/effect-*`, `kit/ns-*`, `facade/*`, `facade-*`, `layer-node-*`, `layer-graph`, `core` y ramas `core-*`, además de variantes claramente relacionadas con services, InstanceState, runtimes y dependency inversion. Dado que existen decenas de ramas históricas solapadas, el análisis profundo se concentra en commits representativos de cada generación y usa el resto como evidencia de cobertura/evolución, no como prueba autónoma de intención.

## 1. Effect services y migración incremental

**Hecho observado — confianza alta.** La búsqueda descubre una familia extensa: `effect/config-paths-service`, `effect/define-service-helper`, `effect/session-transport-service`, `effect/unwrap-run-runtime-facades`, `effect/kill-tuiconfig-installation-facades`, `effectify-*`, `kit/effect-services*`, `kit/effectify-*`, `kit/app-runtime*`, `kit/effect-scoped`, `kit/remaining-instance-state` y ramas relacionadas.

Ramas inspeccionadas en detalle:

- `effect/config-paths-service`
- `effect/define-service-helper`
- `effect/session-transport-service`
- `effect/unwrap-run-runtime-facades`
- `effect/kill-tuiconfig-installation-facades`
- `kit/effect-workspace`
- `kit/effectify-plugin`

**Inferencia arquitectónica — confianza alta.** No constituyen una reescritura única. Son una migración por estrangulamiento: primero se introduce un `Context.Service`/`Layer`, después se conservan facades Promise para callers antiguos y finalmente se eliminan esas facades cuando el composition root puede ejecutar los servicios directamente.

## 2. Namespaces

**Hecho observado — confianza alta.** La familia `kit/ns-*` incluye ramas para Project, Session, Provider, Plugin, Config, Agent, MCP, Tool, File, Storage, Worktree y otros módulos. Se inspeccionaron `kit/ns-project` (`1170776f…`) y `kit/ns-session` (`a83e989f…`). Ambas eliminan `export namespace X { … }`, dejan símbolos al nivel del módulo y añaden barrels (`export * as X from ...`). Los `Service`, `Interface` y `Layer` sobreviven.

**Inferencia arquitectónica — confianza alta.** El namespace TypeScript era organización nominal, no aislamiento real. El boundary estable pasó a ser el módulo/barrel y, para comportamiento con dependencias, el Effect Service.

## 3. Facades y runtimes

Familias observadas:

- `facade/config`
- `facade/file`
- `kit/cleanup-facade`
- `kit/cleanup-facades`
- `kit/cleanup-more-facades`
- `kit/remove-service-facades`
- `kit/remaining-facades`
- `kit/lean-on-facades`

`facade/config` (`591e197c…`) desacopla tests de `AppRuntime` construyendo `Config.defaultLayer` localmente. `effect/kill-tuiconfig-installation-facades` (`9f290d43…`) elimina `makeRuntime` y facades async de `TuiConfig`, `Installation` y `SessionCompaction`.

**Inferencia arquitectónica — confianza alta.** Las facades fueron compatibilidad de transición. La dirección final evita un runtime oculto por módulo y favorece un composition root compartido o layers explícitos en tests.

## 4. `LayerNode` y grafo de dependencias

Ramas inspeccionadas:

- `layer-node-cproject` (`ad4876d3…`)
- `layer-node-integrate` (`b913f854…`)
- `layer-node-location` (`ed3b2a0d…`)
- `layer-node-locfs` (`7d344876…`)
- `layer-node-sprompt` (`d1366ed0…`)
- `layer-graph`

**Hecho observado — confianza alta.** Los tests dejan de reconstruir `Layer.provide`/`Layer.mergeAll` manualmente y pasan a `LayerNode.buildLayer`/`compile` con `LayerNode.replace` para DB, SessionExecution, Credential o Location.

**Inferencia arquitectónica — confianza alta.** Esta familia es la radiografía más fiable de los boundaries reales: si un componente posee `node`, declara deps y puede reemplazarse sin reconstruir vecinos, está siendo tratado como unidad arquitectónica.

## 5. Extracción de `core`

Ramas inspeccionadas:

- `core` (`a0f7731e…`)
- `core-node-guidance` (`3ab16657…`)
- `core-node-consumers` (`12395c37…`)
- `core-project-relocation` (`6f4934ef…`)
- `clean-core-deps` (`52db6391…`)

`core-node-guidance` formaliza que `packages/core` debe ejecutar en Node y no depender de globals/imports exclusivos de Bun. `core-node-consumers` enruta aggregate layers por `LayerNode.compile(node)`.

**Inferencia arquitectónica — confianza alta.** `core` no es una carpeta de utilidades: es el intento de separar dominio/runtime reusable de la aplicación Bun/CLI/server y hacerlo consumible por desktop/SDK/otros hosts.

## 6. Boundaries de plugins y adapters

- `effect-plugin-adapter` (`2bc9f6ea…`): “Compile Effect-authored plugins to the runtime-neutral Promise and Standard Schema plugin contract”.
- `plugin-effect-runtime` (`b7d0582a…`): el entrypoint Effect posee su propia instancia de Effect/Schema y adapta schemas a Standard Schema en el borde.
- `kit/effect-workspace-adapters` (`38a44903…`): separa `PluginWorkspaceAdapter` del `InternalWorkspaceAdapter` Effect.

**Inferencia arquitectónica — confianza alta.** Los plugins son un boundary externo real: OpenCode evita filtrar identidad/runtime de Effect al contrato portable y usa adapters para convertir semántica interna Effect a contratos Promise/Standard Schema.

## 7. Configuración como dependencia

`config-effect-v2` (`0552458f…`) mueve DB placement y configuración de WebSearch desde lecturas `process.env` capturadas en import-time a `Effect.Config`, permitiendo reemplazo mediante `ConfigProvider` durante la construcción del layer.

**Inferencia arquitectónica — confianza alta.** La configuración deja de ser un singleton implícito del proceso y se convierte en dependency injection, lo que mejora testabilidad y permite hosts con proveedores de configuración distintos.

## Línea evolutiva resumida

1. convertir módulos a servicios Effect;
2. mantener facades para compatibilidad;
3. aplanar namespaces y consolidar barrels;
4. eliminar runtimes/facades locales;
5. declarar dependencias con `LayerNode`;
6. formalizar lifetimes global/location/instance/operation;
7. extraer `core` reusable;
8. aislar contratos externos con adapters runtime-neutral.

**Resultado:** el diseño actual es producto de varias generaciones, no de un único refactor. Las ramas descartadas siguen siendo valiosas porque revelan qué dependencias se intentaron cortar y cuáles terminaron codificadas en `dev`.