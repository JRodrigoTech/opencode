# Instrucciones: discovery, herencia y contexto dinámico

## Corrección de auditoría

En `dev` coexisten dos mecanismos de instrucciones. La implementación de `packages/opencode/src/session/instruction.ts` **sigue vigente y está cableada por `SessionPrompt`**; no es correcto describirla como un runtime histórico ya sustituido. En paralelo, Core V2 contiene `packages/core/src/instruction-context.ts`, que integra instrucciones en `SystemContext`/`SessionContextEpoch`.

## Path de producto: `Instruction.Service`

Archivo: `packages/opencode/src/session/instruction.ts`.

### Fuentes globales

El servicio construye `globalFiles` con:

- `<global-config>/AGENTS.md`;
- `~/.claude/CLAUDE.md` como fallback cuando Claude prompt compatibility no está deshabilitada.

`systemPaths()` recorre esos candidatos en orden y se queda con el primer fichero existente.

### Fuentes de proyecto

Los nombres reconocidos son:

1. `AGENTS.md`;
2. `CLAUDE.md` cuando la compatibilidad está habilitada;
3. `CONTEXT.md` marcado explícitamente como deprecated.

Para cada tipo se usa `findUp()` desde el directorio de la instance hasta el worktree. **El primer tipo con matches gana**, por lo que no se apilan automáticamente `AGENTS.md` y `CLAUDE.md` como dos familias simultáneas.

Cuando `OPENCODE_DISABLE_PROJECT_CONFIG` está activo, se evita el discovery de proyecto.

### Instrucciones configuradas

`config.instructions` admite:

- rutas/patrones locales relativos;
- rutas absolutas;
- `~/...`;
- URLs `http://` / `https://`.

Las rutas locales se resuelven mediante glob; las URLs se descargan con timeout y tolerancia a fallo. `system()` devuelve cada contenido prefijado por `Instructions from: <source>`.

### Instrucciones path-local al leer archivos

`resolve(messages, filepath, messageID)` camina desde el directorio del fichero leído hacia la raíz de la instance. Puede cargar archivos de instrucciones cercanos que no formen ya parte de las instrucciones de sistema.

La deduplicación tiene tres defensas:

- no volver a adjuntar paths ya incluidos en `systemPaths()`;
- no volver a adjuntar paths detectados en metadata `loaded` de resultados previos de `read`;
- usar un conjunto de `claims` por `messageID` para evitar duplicados durante el mismo assistant message.

`clear(messageID)` elimina esas claims al terminar el intento.

**Conclusión:** el runtime de producto sigue usando discovery directo de archivos/URLs y una capa de deduplicación ligada a lecturas.

## Cómo entra al system prompt de producto

En `packages/opencode/src/session/prompt.ts`, cada provider turn obtiene en paralelo:

- `SystemPrompt.skills(agent)`;
- `SystemPrompt.environment(model)`;
- `Instruction.system()`;
- `SystemPrompt.mcp(agent, session.permission)`.

Esos elementos forman `system[]` y se entregan al processor/LLM. Por tanto, el contenido de `Instruction.system()` se reconstruye para el turno; no depende de `SessionContextEpoch` en este path.

## Path V2/Core: `InstructionContext`

Archivo: `packages/core/src/instruction-context.ts`.

En el runner V2, las instrucciones ambientales participan como fuente de `SystemContext`. Ese modelo permite:

- identidad de fuente;
- snapshot por sesión;
- refresh/diff;
- updates `system` posteriores al baseline;
- distinguir una observación `unavailable` de una eliminación real.

Esta arquitectura es más durable que el ensamblado directo del path legacy-compatible, pero ambos mecanismos existen en el mismo `dev`.

## Evolución histórica

### `adjust-instructions-logic`

Commit `f2e1dbda16795e61c6533bd5cb1e842aa293604d`.

Explora precedencia/global fallbacks y resolución de instrucciones. Varias de esas preocupaciones siguen visibles en `Instruction.Service`; no debe presentarse toda la familia como obsoleta conceptualmente.

### `read-instruction-dedup`

Commit `0ed6a9c1a9021b1355a931481499049248e8f43f`.

Ataca la doble exposición de una instrucción obtenida mediante `read`. La implementación vigente conserva explícitamente mecanismos de deduplicación (`extract(messages)` + `claims`).

### `instruction-read-race`

Commit `1bbe16b93e2d36f816811629ee2791d73c960d15`.

La preocupación por distinguir una lectura transitoriamente fallida de una retirada real reaparece de forma más explícita en `SystemContext.unavailable` del pipeline V2.

### `namespace-instructions`

Sigue siendo evidencia de branch; no debe describirse como contrato general de `dev` salvo que el código vigente de Code Mode lo confirme específicamente.

## Precedencia: no hay una única tabla global

En el path `packages/opencode`, la precedencia efectiva emerge de varias capas: prompt/provider base, entorno, instrucciones, MCP/skills, agente, transformaciones y el historial. En V2, `agent.info.system`, baseline de `SystemContext` y updates del historial forman otro esquema.

Por ello no es seguro publicar una única lista de precedencia que mezcle ambos pipelines.

## Conclusión

La evolución real no es “el viejo InstructionPrompt desapareció y todo pasó a ContextEpoch”. La verdad de `dev` es:

```text
PRODUCTO / packages/opencode
filesystem + config + URLs -> Instruction.Service -> system[] por turno

V2 / packages/core
instruction source -> SystemContext -> SessionContextEpoch -> baseline/deltas
```

Ambos son comportamiento real y deben distinguirse al documentar instrucciones.