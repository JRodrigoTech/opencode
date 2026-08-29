# Inventario histórico de branches MCP/ACP

## Criterio de inventario

Este archivo registra **refs y commits relevantes observados durante el análisis**, no asume que todo nombre que contenga `mcp` o `acp` constituye una generación arquitectónica independiente.

La fuente autoritativa para comportamiento vigente es `dev@dc4449df0d52199704ea4989a5a993ebbc605612`.

## Calidad de evidencia

| Nivel | Significado |
|---|---|
| A | branch/ref y commit resolubles; diff/código inspeccionado |
| B | branch viva observada, propósito apoyado por nombre/paths pero diff no aislado completamente |
| C | ref aparecido en índice/búsqueda pero no resoluble como branch viva; solo se usa como pista histórica |

## MCP: línea temprana / `v2`

### `mcp-core-skeleton`

- Evidencia: **B**.
- Línea: temprana.
- Observación: branch histórica divergente respecto a `dev`; el diff directo arrastra evolución general del repositorio.
- Uso en la reconstrucción: evidencia de la fase en que MCP estaba más próximo a `packages/core/src/mcp/` y aún no tenía la separación final servicio/catalog/auth/API.

### `mcp-adjustment-work`

- Evidencia: **B**.
- Línea: ajustes sobre la generación temprana.
- Observación: igual que otras branches longevas, no se atribuyen todos sus cambios a MCP solo por comparar HEAD contra `dev`.

### `mcp-prompts`

- Evidencia: **A**.
- HEAD/commit inspeccionado: `dd8e44ad8cb1b864fd62143454ff372a4e3c9fa1`.
- El commit inspeccionado es un merge de `origin/v2` dentro de `mcp-prompts`, con conflicto en `packages/core/src/mcp/index.ts`.
- La línea contiene integración MCP dentro de la arquitectura `v2`, incluyendo administración CLI y evolución de prompts/integrations.
- Clasificación: **generación v2, no comparable limpiamente con `dev` mediante diff bruto**.

### Otras branches históricas `mcp-*`

Durante el inventario se observaron múltiples nombres MCP de vida larga. Cuando su HEAD actual había absorbido cambios posteriores o no representaba ya el propósito nominal, se evitó convertir el nombre en evidencia funcional.

Ejemplo claro: `mcp-attachments` apuntaba a un HEAD cuyo cambio semántico observable era posteriormente `refactor(tui): simplify MCP catalog refresh`, no una implementación autónoma de attachments.

## MCP: capabilities y catálogo

### `feat/mcp-resource-list-changed`

- Evidencia: **A**.
- SHA: `c03c2be4834844e56db5e661bdf9821f58dd3b4a`.
- Commit: `feat(mcp): support resource list change notifications`.
- Añade `ResourceListChangedNotificationSchema` y evento OpenCode `mcp.resources.changed`.
- Solo registra handler si el servidor anuncia `resources.listChanged`.
- Evita publicar eventos de clients que ya no son la conexión activa.
- Estado frente a `dev`: concepto consolidado.

### `feat/mcp-resource-updated`

- Evidencia: **B**.
- SHA observada en branch listing: `742371fdbd4877aa0e6ace6db7fdc2ad530b3443`.
- Clasificación: línea de evolución de resources/events; se agrupa con resource catalog/list change, no como arquitectura independiente.

### `feat/mcp-add-inline-args`

- Evidencia: **B**.
- SHA observada: `7d97c4c0405c2955ba90e2b966073bfaafcb0917`.
- Clasificación: UX/configuración CLI de MCP; no cambia el modelo fundamental del protocolo.

### `kit/mcp-tolerate-bad-output-schemas`

- Evidencia: **B**.
- SHA observada: `ff6f032faade3ef1efad2fcc31450d5cdcd3006b`.
- Propósito nominal y código vigente: tolerancia/normalización ante output schemas MCP defectuosos.
- Clasificación: hardening de boundary externo.

## MCP: lifecycle, auth y seguridad

### `refactor/mcp-connection-results`

- Evidencia: **A**.
- SHA: `2a25bdce74b9429bac8581c286cd7ba343a876e4`.
- Commit: `refactor(mcp): use tagged connection results`.
- Cambio: resultados tagged `connected` / `unavailable`, con status semántico.
- Estado frente a `dev`: refleja la formalización del state machine de conexión.

### `restrict-mcp-env`

- Evidencia: **A**.
- SHA: `8f62645677a919e4ac318091c526f7ced983c2d4`.
- Commit: `fix(opencode): make mcp env handling portable`.
- Cambio: subprocess MCP local recibe baseline de entorno reducido + variables explícitas; corrige case-insensitivity en Windows.
- Estado frente a `dev`: hardening consolidado.

### OAuth refresh cancelable

- Evidencia: **A** mediante commit semántico.
- SHA: `ecf550c88cdadce1f8bb4bd6bb47a09c9899a867`.
- Commit: `fix(mcp): cancel timed out oauth refresh`.
- Cambio: `AbortSignal` atraviesa discovery y refresh OAuth; el timeout cancela I/O real.
- Estado frente a `dev`: consolidado.

### OAuth callback portable

- Evidencia: **A** mediante commit semántico.
- SHA: `4d4eb7de28a355c4a8ac688c0b5d7a29b7f8bea9`.
- Commit: `test(mcp): avoid fixed oauth callback ports`.
- Demuestra lifecycle del callback loopback y redirect URIs dinámicas.

### `elicitation-timeout`

- Evidencia: **A**.
- SHA: `b712f3bb27019f50ff7ffd49c3a0f048aa4a41d1`.
- Commit real: `fix(core): use long mcp tool timeout`.
- Cambio: timeout extremadamente largo para `client.callTool`, con reset por progreso.
- **No es ACP elicitation.** Se clasifica como MCP tool-call hardening.

## MCP: resources como API de plataforma

### Commit `e1d1352d`

- Evidencia: **A**.
- SHA: `e1d1352d8fc001c79086ee7f46508731bce2b8b3`.
- Commit: `feat(mcp): expose resource APIs`.
- Añade resource catalog/read al API/SDK, schemas y `mcp.resources.changed`.
- Estado frente a `dev`: representa el paso de resources desde feature interna a contrato público OpenCode.

### Commit `4d3ff368`

- Evidencia: **A**.
- SHA: `4d3ff36869664bd1e26d67a03fa7f2b3f515e97d`.
- Commit: `refactor(tui): simplify MCP catalog refresh`.
- Cambio: simplifica refresh del catálogo ante eventos.
- Útil para entender consumo frontend, no como nueva generación de wire protocol.

## ACP: línea `nxl-acp-*`

### `nxl-acp-v1`

- Evidencia: **A**.
- HEAD observado: `bb84f7133828ea5ef96399d1bd8f394b4080f328`.
- Comparación con `dev`: branch histórica con **1 commit propio** desde su merge-base y ~498 commits de `dev` posteriores en el momento del análisis.
- Clasificación: primera generación de la fachada ACP moderna.

### `nxl-acp-lifecycle`

- Evidencia: **A**.
- HEAD observado: `26d3f2f1e5166e0d7bd46d8b045c0830e234a63e`.
- Comparación con `dev`: **2 commits propios** desde el mismo merge-base usado por `nxl-acp-v1`.
- Afecta Agent/Event/Permission/Service/tests.
- Clasificación: segunda etapa incremental de la misma familia, centrada en lifecycle y eventos.

### `nxl-acp-elicitation` / commit de elicitation

- Evidencia: **A para el commit, C para el ref durante la consulta**.
- Commit resoluble: `a36c8392b20e306c0688280b9377500a43b486d2`.
- Mensaje: `feat(cli): support acp elicitation`.
- El fetch directo del archivo usando el nombre `nxl-acp-elicitation` no resolvió como branch viva durante el análisis.
- Implementa `unstable_createElicitation` y mapping `form.created` ↔ ACP form elicitation.
- Estado frente a `dev`: la implementación equivalente no aparece en la baseline observada; se clasifica como experimental/no consolidada.

## ACP: configuración y sesiones

### `acp-config-commit`

- Evidencia: **B**.
- Branch viva observada.
- Clasificación: evolución específica de configuración ACP; se agrupa con la consolidación del Service/configOptions en lugar de tratarla como nuevo runtime.

### `feat/acp-session-loading`

- Evidencia: **C/B limitada**.
- El nombre apareció durante búsquedas históricas, pero no se usó como prueba primaria porque la resolución del ref no fue suficientemente estable durante el análisis.
- La capacidad de load/resume sí está demostrada directamente en `dev` mediante `ACPService`.

### `acp-pager`

- Evidencia: **C**.
- Apareció en índice de búsqueda, pero el endpoint de branch no lo resolvió.
- No se usa para atribuir una feature concreta.

## ACP: subagentes

### `acp-subagent-events`

- Evidencia: **A**.
- SHA: `ca357ee2a0e1ac363d03b9017181af7f842e9409`.
- Commit: `fix(acp): surface subagent activity`.
- Comparación con `dev`: un commit propio desde merge-base `4a57013cf8cb163f58638273fd9da8538cd33cb7`, con `dev` muy por delante temporalmente.
- Añade child-session registry, resolución a root session, metadata `opencode/child-session`, prefijos de tool call y routing de permisos.
- Estado frente a la baseline: evolución propuesta no asumida automáticamente como presente en `dev`.

## Branches adyacentes que NO deben confundirse con MCP/ACP

### `reconnect-backoff`

- Evidencia: **A**.
- SHA: `d72fadea15607ac5a36a647a810f47b9b8c21451`.
- Commit: `fix(client): back off event stream reconnection attempts`.
- Modifica `packages/client/src/solid/connection.ts`, no el transport MCP.
- Relevancia: afecta la robustez del event stream que consumen clientes; no demuestra reconnect MCP.

### `prompt-attachments`

- Evidencia: **A**.
- SHA: `237510245789418b470b7c66290b5e63360616fb`.
- Commit: `fix(core): normalize directory attachment paths`.
- Modifica Session attachments internos.
- Relevancia: contexto del modelo de contenido que ACP debe adaptar; no es una branch ACP.

### `protocol-events`

- Evidencia: **A**.
- SHA: `99ce2341189e8eb7503fdec4504bf6b0b8721994`.
- Commit: `refactor(schema): share server event assembly`.
- Modifica el manifiesto de eventos compartido del servidor.
- Relevancia: refuerza el boundary event-driven consumido por ACP, pero no implementa ACP.

## Discrepancias del índice de GitHub

### Hecho confirmado

La búsqueda indexada de branches y el listado/endpoint vivo no siempre coincidieron durante el análisis. Se observaron al menos dos clases de discrepancia:

1. nombres devueltos por búsqueda que ya no resolvían como branch viva (`acp-pager`);
2. branches vivas encontradas por listado paginado que no aparecieron en la primera búsqueda nominal (`acp-subagent-events`).

### Regla aplicada

Una branch no recibe peso de evidencia A únicamente por aparecer en search. Debe resolver o tener un commit concreto resoluble cuyo diff demuestre el comportamiento.

## Agrupación final

| Familia | Generación / propósito |
|---|---|
| `mcp-core-*`, `mcp-adjustment-*`, `mcp-prompts` | MCP temprano integrado en core / línea v2 |
| resources branches + `e1d1352d` | resources, templates, eventos y API pública |
| OAuth/lifecycle/hardening | auth persistente, refresh cancelable, states tagged, seguridad de entorno |
| `nxl-acp-v1` → `nxl-acp-lifecycle` | nacimiento y lifecycle del adapter ACP |
| `acp-config-*` | consolidación de config/session projection |
| elicitation commit | experimento de forms ACP, no consolidado en baseline |
| `acp-subagent-events` | experimento/evolución para child sessions y subagentes |

## Conclusión metodológica

Las generaciones reales se identifican por **merge-base + commits semánticos + paths modificados + comportamiento en `dev`**, no por similitud del nombre de branch. Esta regla es especialmente necesaria en este repositorio porque numerosas branches permanecieron abiertas durante grandes reescrituras y merges de `v2`/`dev`.