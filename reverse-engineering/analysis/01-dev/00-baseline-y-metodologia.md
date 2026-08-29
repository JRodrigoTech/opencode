# 00 — Baseline y metodología

## Baseline

**[CONFIRMADO]** Este análisis toma como arquitectura vigente `dev@dc4449df0d52199704ea4989a5a993ebbc605612`, observado el 29 de agosto de 2026.

La documentación se escribe exclusivamente en la branch `reverse-engineering`; `dev` se utiliza sólo como fuente de lectura.

## Método de reverse engineering

El análisis se realizó por **flujos y boundaries**, no por listado de archivos:

1. localizar composition roots y entrypoints;
2. reconstruir el flujo de un prompt desde cliente hasta modelo y vuelta;
3. identificar ownership de estado y scope de servicios;
4. seguir writes/eventos hasta persistencia;
5. seguir la frontera server/API hacia TUI y Desktop;
6. identificar coexistencia entre implementaciones legacy y nuevas;
7. separar contratos observados de interpretaciones evolutivas.

## Evidencias de alto nivel

**[CONFIRMADO]** El root `package.json` declara un workspace `packages/*` y utiliza Bun. El paquete `packages/opencode` sigue siendo un ejecutable/composition root de gran peso y depende de paquetes workspace especializados.

**[CONFIRMADO]** `packages/opencode/package.json` depende, entre otros, de `@opencode-ai/core`, `@opencode-ai/llm`, `@opencode-ai/server`, `@opencode-ai/tui`, `@opencode-ai/schema` y `@opencode-ai/protocol`.

**[CONFIRMADO]** `packages/core/package.json` contiene primitives transversales de runtime, Effect layers, database, session runner y adaptadores Bun/Node para SQLite/PTY/filesystem.

**[CONFIRMADO]** `packages/llm/package.json` expone providers y protocolos explícitos: Anthropic Messages, Bedrock Converse, Gemini, OpenAI Chat, OpenAI-compatible Chat y OpenAI Responses.

**[INFERENCIA]** La arquitectura de paquetes está siendo usada como mecanismo de extracción de boundaries desde `packages/opencode`, pero el composition root legacy continúa orquestando una parte relevante de esos servicios.

## Criterio para hechos e inferencias

Se marca como **CONFIRMADO** aquello que puede demostrarse directamente por imports, tipos, llamadas, schemas, layers, queries SQL o control flow. Se marca como **INFERENCIA** cualquier conclusión sobre intención de diseño, dirección futura o estrategia de migración, aunque esté respaldada por múltiples indicios.

## Limitaciones

- No se ejecutó instrumentación dinámica ni profiling.
- No se compararon en este agente branches históricas; esa tarea pertenece a los otros análisis del proyecto.
- La existencia de paths `v1`, `v2`, bridges o flags no implica por sí sola cuál será el estado final; sólo permite describir coexistencia y compatibilidad.

## Identificadores reproducibles

Commit `dev`: `dc4449df0d52199704ea4989a5a993ebbc605612`.

Commit de `reverse-engineering` antes de este análisis: `f540bc575606064912a589fb713fcbc8ac726ea4`.
