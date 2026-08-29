# 04 — Prompt, contexto y compaction

## Alcance y regla de lectura

Este análisis reconstruye cómo OpenCode prepara el contexto del modelo y cómo reduce historia cuando se aproxima al límite de contexto.

La baseline es `dev@dc4449df0d52199704ea4989a5a993ebbc605612`.

### Corrección de auditoría

En `dev` **coexisten dos pipelines distintos** y no deben describirse como si fueran uno solo:

1. **Pipeline de producto actualmente compuesto por `packages/opencode`**: `SessionPrompt` + `SystemPrompt` + `Instruction` + `MessageV2` + `SessionCompaction` + `SessionProcessor` + `LLM`.
2. **Pipeline V2/Core implementado en `packages/core`**: `SessionRunner` + `SystemContext` + `SessionContextEpoch` + `SessionHistory` + compaction V2 + `@opencode-ai/llm`.

La existencia del segundo en `dev` demuestra una arquitectura V2 real, pero **no demuestra que haya sustituido al primero como path por defecto del producto**. El composition root de `packages/opencode/src/server/routes/instance/httpapi/server.ts` sigue incluyendo `SessionPrompt`, `SessionCompaction`, `SessionProcessor` y `LLM`.

Esta distinción corrige una versión anterior de esta documentación que etiquetaba varias propiedades exclusivas del runner V2 como “el runtime actual” sin matiz.

## Pipeline de producto: `packages/opencode`

El loop efectivo de `packages/opencode/src/session/prompt.ts`:

1. carga el historial mediante `MessageV2.filterCompactedEffect()`;
2. determina el último usuario/modelo/agente;
3. materializa tools;
4. obtiene en paralelo:
   - `SystemPrompt.skills(agent)`;
   - `SystemPrompt.environment(model)`;
   - `Instruction.system()`;
   - `SystemPrompt.mcp(agent, session.permission)`;
   - `MessageV2.toModelMessagesEffect(...)`;
5. entrega `system`, mensajes, tools, modelo y permisos a `SessionProcessor`;
6. el processor consume el stream de `LLM.Service`;
7. si el resultado exige compaction, crea una compaction con `packages/opencode/src/session/compaction.ts` y continúa el loop.

`Instruction.Service` está por tanto **vigente y cableado**. Resuelve `AGENTS.md`, `CLAUDE.md`, `CONTEXT.md` deprecated, instrucciones configuradas y URLs; además resuelve instrucciones path-local durante lecturas evitando duplicados.

`SystemPrompt.Service` también está vigente: construye contexto de entorno, catálogo de skills e instrucciones MCP, mientras otras capas añaden prompts/provider transforms.

## Pipeline V2/Core

`packages/core/src/session/runner/llm.ts` implementa otra arquitectura:

- `SystemContextRegistry` combina fuentes de contexto;
- `SessionContextEpoch` mantiene baseline/snapshot por sesión;
- `SessionHistory.entriesForRunner()` selecciona la historia desde la frontera apropiada;
- `toLLMMessages()` proyecta mensajes V2;
- `SessionCompaction` puede compactar antes del turno y recuperar un overflow previo a salida assistant;
- `LLMClient` ejecuta el request portable.

En ese pipeline, `request.system` combina `agent.info?.system` con `system.baseline`, y los cambios posteriores de contexto pueden aparecer como mensajes `system` ordenados en la historia.

Esto es **comportamiento confirmado del runner V2 presente en `dev`**, no una descripción universal del `SessionPrompt` legacy-compatible.

## Compaction: dos implementaciones

### Path `packages/opencode`

`packages/opencode/src/session/compaction.ts` es la compaction utilizada por `SessionPrompt`.

Hechos relevantes:

- usa el agente `compaction`;
- selecciona una cola reciente por turns;
- el presupuesto reciente por defecto es aproximadamente el 25 % del contexto utilizable, limitado entre 2.000 y 15.000 tokens, salvo configuración;
- trunca resultados de tools a 2.000 caracteres al construir el material de resumen;
- reutiliza `buildPrompt()` del core, pero conserva lifecycle, mensajes y processor del path legacy-compatible;
- puede podar outputs antiguos de tools (`PRUNE_MINIMUM=20_000`, `PRUNE_PROTECT=40_000`).

### Path V2/Core

`packages/core/src/session/compaction.ts` implementa la política V2. Sus defaults y su representación durable no deben trasladarse automáticamente al path anterior.

Entre sus propiedades están el checkpoint V2, una cola reciente explícita y el recovery acotado de overflow integrado con `SessionRunner`.

## Overflow

También hay dos políticas.

En el path `packages/opencode`, `packages/opencode/src/session/overflow.ts` calcula el contexto utilizable a partir de `model.limit.*`, `compaction.reserved` y el output máximo; si `compaction.auto === false` no dispara auto-compaction. El loop legacy decide compactar a partir del usage/resultado procesado.

En el runner V2, la compaction puede ejecutarse proactivamente sobre el request ensamblado y existe además recovery reactivo de `context-overflow` **sólo antes de que haya empezado salida assistant**, con un único reintento tras reconstruir el request.

No es correcto atribuir ese recovery V2 al `SessionPrompt` por defecto sin especificar el pipeline.

## Documentos

- [`01-system-context/README.md`](01-system-context/README.md): `SystemContext`/`SessionContextEpoch` y su relación con el pipeline V2, contrastados con el path activo de `packages/opencode`.
- [`02-instructions/README.md`](02-instructions/README.md): discovery de instrucciones vigente y arquitectura V2 de instruction context.
- [`03-compaction/README.md`](03-compaction/README.md): las dos implementaciones de compaction y sus diferencias.
- [`04-overflow-limits-retry/README.md`](04-overflow-limits-retry/README.md): overflow legacy-compatible frente a recovery V2.
- [`05-prompt-pipeline-continuity/README.md`](05-prompt-pipeline-continuity/README.md): ambos pipelines de construcción/continuidad.
- [`06-branch-inventory/README.md`](06-branch-inventory/README.md): inventario histórico de branches.

## Referencias primarias de `dev`

### Path de producto

- `packages/opencode/src/session/prompt.ts`
- `packages/opencode/src/session/system.ts`
- `packages/opencode/src/session/instruction.ts`
- `packages/opencode/src/session/compaction.ts`
- `packages/opencode/src/session/overflow.ts`
- `packages/opencode/src/session/processor.ts`
- `packages/opencode/src/session/llm.ts`
- `packages/opencode/src/server/routes/instance/httpapi/server.ts`

### Path V2/Core

- `packages/core/src/session/runner/llm.ts`
- `packages/core/src/session/runner/to-llm-message.ts`
- `packages/core/src/session/history.ts`
- `packages/core/src/session/compaction.ts`
- `packages/core/src/session/context-epoch.ts`
- `packages/core/src/system-context/index.ts`
- `packages/core/src/instruction-context.ts`

## Conclusión

La verdad arquitectónica de `dev` es **híbrida**. OpenCode no puede documentarse correctamente diciendo simplemente “usa SessionPrompt” ni diciendo “ya usa ContextEpoch para todo”. El primero sigue siendo un runtime de producto cableado; el segundo es una arquitectura V2 sustancial presente en Core y utilizada por superficies V2. Cualquier afirmación sobre contexto, compaction u overflow debe indicar a cuál de los dos pipelines se refiere.