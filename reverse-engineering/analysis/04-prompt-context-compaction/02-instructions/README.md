# Instrucciones: discovery, herencia y variantes dinámicas

## Resumen

La evolución de instrucciones muestra dos arquitecturas distintas:

1. **runtime histórico:** resolver archivos de instrucciones y añadir/injectar su contenido directamente al prompt o al historial;
2. **`dev` actual:** representar las instrucciones ambientales como una fuente durable de `SystemContext`, de modo que la sesión conserva qué versión de esas instrucciones conoce el modelo y recibe deltas cuando cambian.

## Estado vigente en `dev`

### Fuente de instrucciones

**CONFIRMADO EN `dev`:** `packages/core/src/instruction-context.ts` integra las instrucciones del workspace/proyecto en `SystemContext`. Su identidad de fuente permite snapshot, refresh y updates.

La consecuencia importante no es sólo dónde se lee `AGENTS.md`, sino cómo se trata temporalmente:

- el contenido inicial forma parte del baseline privilegiado;
- cambios posteriores se convierten en updates `system`;
- una lectura temporalmente fallida no debe interpretarse como eliminación;
- tras una compaction el epoch puede rebaselinar el estado conocido.

### Instrucciones del agente

**CONFIRMADO EN `dev`:** el agente seleccionado puede aportar `agent.info.system`. El runner lo coloca en `request.system` antes/junto al baseline del `SessionContextEpoch`.

Por tanto hay, como mínimo, dos clases de instrucciones privilegiadas:

- instrucciones definidas por la configuración/selección del agente;
- instrucciones/contexto ambiental producido por fuentes registradas.

### Skills y referencias como contexto dinámico

**CONFIRMADO EN `dev`:** `packages/core/src/session/runner/llm.ts` combina `systemContext.load()`, `skillGuidance.load(agent)` y `referenceGuidance.load()` antes de inicializar/preparar el context epoch. Esto significa que la arquitectura de fuentes dinámicas ya no se limita a `AGENTS.md`: skills y guidance de referencias pueden participar en el mismo baseline/diff contextual.

## Evolución histórica de `AGENTS.md`

### `adjust-instructions-logic`

**CONFIRMADO EN BRANCH** — commit `f2e1dbda16795e61c6533bd5cb1e842aa293604d`.

En la implementación histórica `packages/opencode/src/session/instruction.ts` se ajustó la resolución de archivos globales:

- podían coexistir `AGENTS.md` del directorio de configuración explícito y del config global;
- `~/.claude/CLAUDE.md` actuaba como fallback cuando no existían esos `AGENTS.md` y no estaba deshabilitado;
- las instrucciones configuradas podían resolverse como rutas/globs relativos o absolutos;
- `CONTEXT.md` aparecía todavía como nombre deprecated en esa generación.

**NO VIGENTE COMO IMPLEMENTACIÓN:** esta lógica pertenece al runtime antiguo. Es útil para reconstruir precedencias, pero `dev` ya no usa esta pieza como centro del ensamblado del prompt.

### `instruction-rename`

**CONFIRMADO EN BRANCH, alcance histórico:** esta línea acompaña la transición desde instrucciones tratadas como prompt auxiliar hacia instrucciones con identidad/durabilidad en la sesión. Debe leerse junto con las ramas de instruction discovery posteriores, no como API vigente.

### `read-instruction-dedup`

**CONFIRMADO EN BRANCH** — commit `0ed6a9c1a9021b1355a931481499049248e8f43f`.

Esta etapa atacaba un problema concreto: al leer un `AGENTS.md` path-local mediante una tool, su contenido podía convertirse en instrucción de sesión y además aparecer como output de la propia lectura. La solución histórica deduplicaba esa doble exposición y registraba metadata de paths de instrucciones.

**INFERENCIA, confianza alta:** la necesidad de dedup es evidencia de que el diseño “inyectar instrucciones descubiertas como mensajes sintéticos” tenía acoplamiento fuerte entre tool execution e instruction state. La arquitectura `SystemContext` elimina gran parte de ese acoplamiento al volver las instrucciones una fuente observable independiente.

### `instruction-read-race`

**CONFIRMADO EN BRANCH** — commit `1bbe16b93e2d36f816811629ee2791d73c960d15`.

Corrige una carrera filesystem: una instrucción observada podía desaparecer entre `exists()` y `read`. La branch evita interpretar esa lectura fallida como una retirada real de una instrucción previamente admitida.

**CONFIRMADO EN `dev` COMO SEMÁNTICA GENERAL:** `SystemContext.unavailable` conserva esta distinción entre “no pude observarla” y “la fuente fue eliminada”.

## Instrucciones heredadas y path-local

### Modelo histórico

La generación `InstructionPrompt` hacía discovery ascendente desde el directorio activo hacia el límite del worktree, combinándolo con archivos globales/configurados. Eso producía una jerarquía de instrucciones heredadas por ubicación.

### Modelo actual

**CONFIRMADO EN `dev`:** la jerarquía física sigue siendo responsabilidad del proveedor de instrucciones, pero la sesión ya no necesita asumir que todos esos archivos son un bloque estático. Recibe el estado materializado de la fuente y sus cambios a través del context epoch.

**INFERENCIA, confianza alta:** el boundary correcto es:

```text
filesystem/config -> instruction discovery -> fuente SystemContext -> snapshot por sesión -> baseline/deltas
```

no:

```text
filesystem/config -> concatenación directa al prompt en cada turn
```

## `namespace-instructions`

### Qué introduce

**CONFIRMADO EN BRANCH** — commit base `5d4c4d0ede0a74d38649ce7094c78d55eb4246fa`, seguido por validaciones y simplificación de presupuesto.

La branch añade instrucciones asociadas a namespaces de herramientas de Code Mode:

- una tool puede registrar `namespaceInstructions` junto con su namespace;
- no se admiten instrucciones de namespace si la tool no está namespaced;
- varias tools del mismo namespace no pueden registrar instrucciones conflictivas;
- las instrucciones se renderizan una sola vez antes de los listings del namespace;
- sobreviven aunque el presupuesto sea tan pequeño que ninguna firma de tool quepa;
- **consumen presupuesto** del catálogo;
- si cambian, el catálogo se reemplaza de forma durable para evitar que el modelo mezcle guidance viejo y nuevo.

### Estado frente a `dev`

**NO CONFIRMADO EN `dev` / BRANCH-ONLY:** a 29 de agosto de 2026 esta extensión no debe describirse como contrato del runtime vigente. Es evidencia de una dirección arquitectónica: hacer que instrucciones dinámicas asociadas a subsistemas registrables tengan la misma propiedad de durabilidad y reemplazo que el resto del contexto.

## Precedencia conceptual

El código no expone una única tabla formal de precedencia, pero la construcción del request permite distinguir niveles:

1. `agent.info.system` — instrucciones propias del agente activo.
2. baseline/deltas de `SystemContext` — entorno, instrucciones del proyecto y guidance dinámico.
3. historial de usuario/assistant/tools — contenido conversacional.
4. checkpoint de compaction — historia resumida, explícitamente degradada a role `user` para que no gane autoridad accidentalmente.

**INFERENCIA, confianza alta:** la elección de role del checkpoint es una defensa deliberada contra instruction resurrection: una frase que apareció históricamente dentro de una conversación no debe volver a convertirse en instrucción system sólo porque fue resumida.

## Conclusión

La evolución de instrucciones es una transición desde **resolución de archivos** hacia **gestión de estado instruccional**. El problema principal ya no es sólo “qué `AGENTS.md` leo”, sino “qué instrucciones cree vigentes el modelo en esta sesión y cómo actualizo esa creencia sin duplicación, carreras ni resurrección de instrucciones antiguas”.