# System context y context epoch

## Corrección de alcance

`SystemContext` y `SessionContextEpoch` existen y están implementados en `dev`, pero pertenecen al **pipeline V2/Core**. No son la única ni la principal forma en que `packages/opencode` construye hoy el contexto del modelo.

El composition root del producto sigue cableando `SessionPrompt`, `SystemPrompt` e `Instruction`; por tanto, toda afirmación de este documento debe leerse como **“confirmado en el runner V2 de `dev`”**, no como “único runtime de OpenCode”.

## Pipeline V2/Core

### `SystemContext`

Archivo: `packages/core/src/system-context/index.ts`.

Responsabilidades:

- representar fuentes de contexto mediante identidad estable;
- producir baseline;
- observar/refrescar fuentes;
- calcular updates por fuente;
- distinguir una fuente eliminada de una observación temporalmente `unavailable`.

Una fuente puede representar instrucciones, entorno, proyecto u otro estado operativo.

### Builtins

Archivo: `packages/core/src/system-context/builtins.ts`.

Incluye contexto como working directory/workspace, información Git, plataforma y fecha. La propiedad arquitectónica relevante es que estos datos se modelan como fuentes observables, no como una sola cadena anónima.

### Instrucciones

`packages/core/src/instruction-context.ts` integra instrucciones ambientales en este mismo modelo de fuente/snapshot.

### `SessionContextEpoch`

Archivo: `packages/core/src/session/context-epoch.ts`.

Mantiene por sesión:

- baseline de contexto privilegiado;
- snapshot codificado de las fuentes conocidas;
- frontera de secuencia asociada;
- integración con la historia/compaction V2.

El snapshot representa lo que el modelo ya fue informado, no simplemente el estado físico instantáneo del filesystem.

Antes de un request V2, el runner inicializa/prepara el epoch, compara fuentes y puede persistir updates `system` cronológicos.

### `unavailable` no significa `removed`

Esta distinción está implementada para evitar retirar contexto previamente admitido por una lectura transitoria fallida.

## Cómo entra al request V2

`packages/core/src/session/runner/llm.ts` construye `request.system` con:

```text
agent.info?.system
system.baseline
```

Los updates posteriores al baseline permanecen en la historia proyectada.

## Lo que hace el path de producto de `packages/opencode`

El runtime cableado por `SessionPrompt` utiliza otro mecanismo:

- `SystemPrompt.environment(model)` genera contexto de entorno y referencias;
- `Instruction.system()` lee instrucciones globales/proyecto/configuradas;
- `SystemPrompt.skills(agent)` añade catálogo/guidance de skills;
- `SystemPrompt.mcp(agent, permission)` añade instrucciones MCP;
- `SessionPrompt` concatena esos elementos en el `system` enviado a `SessionProcessor`/`LLM`.

Ese path **no depende de `SessionContextEpoch` para ensamblar el system prompt del turno**.

## Relación evolutiva

Las branches `context-checkpoint`, `system-context` y `feat/core-v2-session-context-epoch` son evidencia clara de la evolución hacia estado contextual durable. Esa arquitectura está materializada en Core V2, pero convive con la construcción legacy-compatible en `packages/opencode`.

## Boundary correcto

Para V2:

```text
fuentes dinámicas -> snapshot conocido por sesión -> baseline + deltas -> request
```

Para el path `packages/opencode`:

```text
config/filesystem/runtime -> SystemPrompt + Instruction -> system[] del turno
```

## Conclusión

La aportación de `SystemContext` es convertir el contexto privilegiado en estado observable y versionable por sesión. Es una arquitectura real en `dev`, pero debe documentarse como **pipeline V2/Core coexistente**, no como reemplazo ya consumado de `SessionPrompt`.