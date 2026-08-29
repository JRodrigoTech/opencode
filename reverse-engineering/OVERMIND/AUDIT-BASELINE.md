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

Para distinguir implementación de intención se respeta el propio orden de autoridad del proyecto:

1. source activo y accepted tests;
2. `AGENTIX/STATE.md` para estado resumido;
3. `AGENTIX/CURRENT_RUNTIME_BASELINE.md` para invariants actuales;
4. contracts especializados de `AGENTIX/*_ARCHITECTURE/` para target design;
5. `FUTURE/`, rationale e implementation history solo como material no autoritativo cuando así se declara.

## Método

Cada recomendación se construyó cruzando:

1. mecanismo equivalente en OpenCode production;
2. implementación real de Overmind;
3. contracts normativos o deferred boundaries ya definidos por Overmind;
4. dependencia necesaria para introducirlo sin romper ownership.

No se presenta como implementado nada que `STATE.md` marque deferred.

## Límites

Este trabajo es arquitectura/reverse engineering estático. No ejecuta la suite de Overmind ni proveedores live. Los tests documentados por Overmind se consideran evidencia del repository baseline, pero no fueron reejecutados desde esta conexión GitHub.

Las propuestas son deliberadamente incrementales: la existencia de una solución en OpenCode no prueba que Overmind deba implementarla inmediatamente.
