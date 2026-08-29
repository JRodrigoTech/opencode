# Genealogía de refactors: integrado, parcial, reemplazado y abandonado

## Método

La clasificación siguiente no presupone que una branch no mergeada sea “fallida”. Para reverse engineering, una branch puede ser extremadamente informativa aunque su implementación concreta haya sido reemplazada. Se compara su idea con estructuras visibles en `dev`.

## 1. Migración a Effect Services — **integrada**

**Evidencia observada — confianza alta.** `dev` contiene `Context.Service`, Layers y nodes en prácticamente todos los subsistemas principales de la shell de aplicación y del core. `AppRuntime` compone esos servicios.

Ramas históricas relevantes: `effectify-*`, `kit/effectify-*`, `effect/config-paths-service`, `kit/effect-services*`.

**Conclusión:** la idea central sobrevivió y se convirtió en base arquitectónica.

## 2. Runtime privado por servicio con `makeRuntime` — **reemplazado en gran medida**

**Evidencia observada — confianza alta.** Ramas tempranas crean facades async backed por runtimes locales; `effect/kill-tuiconfig-installation-facades` y `effect/unwrap-run-runtime-facades` eliminan este patrón de múltiples servicios. `dev` usa `AppRuntime` como composition root principal.

**Conclusión:** fue una técnica de transición útil, no la forma final preferida para servicios internos.

**Excepción:** adapters Promise externos pueden seguir siendo válidos si compilan/usan el node graph canónico, como muestra `core-node-consumers`.

## 3. Namespaces `export namespace` — **reemplazados como boundary**

`kit/ns-project`, `kit/ns-session` y la familia `kit/ns-*` aplanan namespaces y conservan barrels + Services.

**Evidencia observada — confianza alta.** La arquitectura vigente usa con frecuencia `export * as X from ...` como ergonomía, mientras el boundary de DI vive en Service/LayerNode.

**Conclusión:** el namespace no era encapsulación significativa y su eliminación no destruye el componente.

## 4. Facades Promise internas — **eliminación progresiva**

Ramas `facade/*`, `kit/cleanup-*facade*`, `kit/remove-service-facades`, `effect/kill-*facades` muestran una campaña sostenida.

**Conclusión — confianza alta.** La intención consolidada es que callers internos Effect consuman el servicio directamente y que tests construyan layers locales. Las facades quedan reservadas a compatibility/external boundaries.

## 5. `LayerNode` / graph metadata — **integrado y ampliado**

Las ramas `layer-node-*` inicialmente simplifican tests; `dev` contiene una implementación más rica con:

- nodes layer/unbound/group;
- dependency validation;
- replacements;
- tags;
- cycle detection;
- hoisting;
- compilation.

`AppRuntime`, Project y `location-services.ts` lo usan en producción.

**Conclusión — confianza muy alta.** Este refactor pasó de helper de wiring a infraestructura arquitectónica central.

## 6. Tags global/location — **integrados**

`packages/core/src/effect/app-node.ts` codifica la dirección `location -> global` y prohíbe la inversa.

**Conclusión:** es una regla arquitectónica consolidada. `LocationServiceMap` existe para cruzar el boundary sin violarla.

## 7. `LayerNode.buildLayer` de primeras ramas — **API evolucionada, concepto integrado**

Las ramas `layer-node-*` usan APIs históricas como `buildLayer`. La implementación actual usa predominantemente `compile`, `group`, `hoist` y builders de app.

**Conclusión — confianza alta.** La API concreta evolucionó; el concepto de graph declarativo y replacements sobrevivió.

## 8. Layer fresco por operación — **rechazado para SessionTransport**

`effect/session-transport-service` sustituye un diseño con Layer fresh por un factory service + handles por operación.

**Conclusión — confianza alta.** No todo stateful object merece un Layer. El proyecto refinó la frontera entre DI/component lifetime y operation lifetime.

## 9. `InstanceState` — **integrado, pero coexistiendo con Location graph**

La familia `kit/remaining-instance-state`, `kit/instance-state-provide` y las migraciones Effect muestran esfuerzo explícito por mover estado implícito a InstanceState. `dev` todavía lo usa en Config, Plugin, Provider y otros servicios legacy.

En paralelo, core nuevo materializa subgrafos por `Location.Ref`.

**Conclusión — confianza alta.** `InstanceState` no está “eliminado”, pero forma parte de una generación legacy/app que coexiste con una arquitectura location-scoped más declarativa. No hay evidencia suficiente para afirmar que deba desaparecer por completo.

## 10. `core` portable — **integrado, extracción aún parcial**

La regla Node-compatible está formalizada y `packages/core` aloja gran parte de la infraestructura/domain v2. Sin embargo, Provider, Config legacy, Plugin legacy, server/CLI y bridges siguen en `packages/opencode`.

**Conclusión — confianza alta.** La extracción de core es real pero no completa. La shell de aplicación sigue siendo un strangler/compatibility layer necesario.

## 11. Effect público en plugins — **redefinido mediante adapter boundary**

Las ramas de plugin muestran dos movimientos aparentemente opuestos:

1. `plugin-effect-runtime` hace que el entrypoint Effect posea explícitamente su runtime Effect/Schema;
2. `effect-plugin-adapter` evita que ese runtime se convierta en el contrato host universal y compila al boundary Promise + Standard Schema.

**Conclusión — confianza alta.** No se abandonó Effect authoring; se abandonó la idea de compartir implícitamente identidad de Effect entre host y plugin. La solución final es ownership + adaptación.

## 12. Workspace Effect migration — **parcial / históricamente tardía**

`kit/effect-workspace` conserva un checklist donde Workspace y SyncEvent estaban entre los últimos pendientes, mientras otros servicios ya se marcaban migrados. `kit/effect-workspace-adapters` aclara internal vs public adapters.

**Conclusión — confianza media-alta.** Workspace fue un boundary difícil/tardío porque cruza Project, Worktree, plugins y control-plane. La existencia de adapters explícitos indica progreso, pero su historia es más transicional que `LayerNode` o filesystem core.

## 13. Provider modularization — **parcial**

Las familias `effectify-provider`, provider-specific/native-provider branches y `ProviderV2`/`ModelV2` muestran intentos de separación. Sin embargo, el Provider de `packages/opencode` aún coordina catálogo, auth, SDK construction, env/config y transformaciones específicas.

**Conclusión — confianza alta.** El boundary conceptual está identificado, pero la implementación sigue siendo una de las zonas de mayor concentración de responsabilidades.

## 14. Config desde `process.env` import-time — **en proceso de reemplazo**

`config-effect-v2` mueve DB/WebSearch a `Effect.Config` y permite `ConfigProvider`.

**Conclusión — confianza alta.** Existe intención clara de convertir environment/configuration en dependency injectable, pero no todo el código legacy ha sido convertido.

## Matriz de estado

| Línea | Estado | Evidencia |
|---|---|---|
| Effect Services | Integrada | services/nodes en `dev` |
| AppRuntime central | Integrada | `effect/app-runtime.ts` |
| runtime local por módulo | Reemplazado mayormente | eliminación de `makeRuntime` facades |
| TS namespaces como encapsulación | Reemplazado | `kit/ns-*` |
| facades internas Promise | Eliminación progresiva | `facade/*`, `effect/kill-*` |
| LayerNode graph | Integrado/canónico | core + AppRuntime + tests |
| global/location tags | Integrado | `app-node.ts` |
| InstanceState | Integrado legacy/app | Config/Plugin/etc. |
| Location subgraphs | Integrado core | `location-services.ts` |
| fresh Layer por operation handle | Reemplazado en transporte | session transport factory |
| Core Node-compatible | Integrado | `core/AGENTS.md` + consumers |
| plugin Effect identity compartida | Rechazada | Standard Schema adapter |
| Workspace modularization | Parcial | adapter bridge/history |
| Provider decomposition | Parcial | ProviderV2 + monolito app restante |
| Effect.Config | Parcial/en expansión | DB/WebSearch migration |

## Lectura final de los refactors “abandonados”

**Inferencia arquitectónica — confianza alta.** Las ramas no integradas muestran tres clases de experimento:

1. **scaffolding migratorio** que desaparece cuando deja de ser necesario (`makeRuntime` facades);
2. **API experimental reemplazada por una versión más fuerte** (`buildLayer` → graph actual con tags/hoist/compile);
3. **boundary conceptual que sí sobrevive aunque cambie su implementación** (Effect services, core, location scoping, plugin adapters).

Por eso la evidencia más valiosa no es preguntar “¿se mergeó esta branch?”, sino “¿qué dependencia intentó hacer explícita y dónde aparece esa misma frontera en `dev`?”.