# Audit Baseline

## Repositorios congelados

### OpenCode

- repository: `JRodrigoTech/opencode`
- source branch: `production`
- commit: `df35e842f59bc115bb7c0479a8e11f017d443f2c`
- tree: `bf3012d135092dd7caa961147770a7540a0e16e1`
- commit time: `2026-08-28T12:04:58Z`

La arquitectura OpenCode usada aquí es la auditada en `reverse-engineering/OPENCODE_ARCHITETURE/` y revalidada en su `AUDIT-REPORT.md`.

### Overmind

- repository: `JRodrigoTech/Overmind`
- source branch: `main`
- commit: `c8b68b55b4a057232c764cafc545d4633fbeb22f`
- tree: `1bd565b0587e89f0bac43cd39e0aa534252f4ef7`
- commit time: `2026-08-28T23:17:55Z`
- commit message: `docs: add context compiler refactor guardrails`

## Autoridad de Overmind

Se respeta el orden de autoridad definido por el propio proyecto:

1. source activo y accepted tests;
2. `AGENTIX/STATE.md`;
3. `AGENTIX/CURRENT_RUNTIME_BASELINE.md`;
4. contracts especializados `AGENTIX/*_ARCHITECTURE/`;
5. `FUTURE/`, rationale e implementation history según su status.

## Método revisado

La comparación usa dos preguntas separadas:

1. **¿Qué mecanismo demuestra OpenCode?** — hecho de reverse engineering.
2. **¿Debe vivir ese mecanismo dentro de Overmind?** — decisión arquitectónica evaluada contra los principios y necesidades de Overmind.

Un mecanismo puede estar perfectamente implementado en OpenCode y aun así clasificarse como `DELEGATE`, `REFERENCE-ONLY` o `DO-NOT-COPY`.

La segunda revisión verificó además surfaces que permiten usar OpenCode como agente externo:

- `opencode run --format json` y session resume;
- `opencode acp` sobre stdio;
- ACP session/prompt/cancel operations;
- ACP permission forwarding;
- TaskTool/subagents internos.

## Regla contra sesgo de feature parity

No se recomienda una primitive Overmind porque aparezca en OpenCode. La recomendación requiere al menos una de estas condiciones:

- ya existe una necesidad explícita en el roadmap/contracts de Overmind;
- resuelve una necesidad observada en source actual;
- es la primitive mínima necesaria para integrar una capability real;
- su semántica debe ser gobernada universalmente por Core.

En caso contrario se mantiene como referencia o dentro del agente externo.

## Límites

Trabajo estático de arquitectura/reverse engineering. No se ejecutó la suite ni providers live desde esta conexión.

La viabilidad de `run`/ACP se basa en source production y no equivale a una prueba end-to-end Overmind -> OpenCode. Esa integración debe validarse con un spike independiente antes de consolidar contratos.
