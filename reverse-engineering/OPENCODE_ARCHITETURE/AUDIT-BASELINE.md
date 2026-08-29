# Audit Baseline

**Status:** VERIFIED-CODE

## Source commit

- Branch: `production`
- Commit: `df35e842f59bc115bb7c0479a8e11f017d443f2c`
- Commit tree: `bf3012d135092dd7caa961147770a7540a0e16e1`
- Commit timestamp: `2026-08-28T12:04:58Z`
- Commit subject: `docs(zen): add Ling 3.0 Flash Fin Free (#45923)`

No se utiliza el nombre móvil `production` como única referencia de auditoría: el SHA anterior es la baseline congelada de este estudio.

## Documentation branch

La documentación se almacena en la rama `reverse-engineering`, dentro de:

`reverse-engineering/OPENCODE_ARCHITETURE/`

La rama fuente `production` no se modifica.

## Method

1. Fijar el commit de producción y su tree SHA.
2. Inventariar packages y directorios del runtime.
3. Inspeccionar los orquestadores y contratos de mayor fan-in/fan-out.
4. Seguir flujos de control completos: input → prompt → loop → LLM → processor → tools → persistencia.
5. Seguir flujos de estado: session/message/part, permisos, snapshots, compaction y cancelación.
6. Identificar seams de adaptación: providers, AI SDK/native LLM, MCP, plugins, ACP y HTTP.
7. Documentar invariantes y precedencias, no solo nombres de clases.
8. Registrar blob SHA de fuentes críticas para detectar drift futuro.

## Evidence policy

Una descripción se considera auditada cuando puede señalar al menos una fuente de código y no contradice las demás fuentes del mismo flujo. Para decisiones de comportamiento se priorizan implementaciones sobre tipos o README.

Los README de packages se aceptan como evidencia solo para intención o boundary contractual; para comportamiento se cruza con código.

## Production/dev equality used during analysis

Varios hotspots previamente leídos en `dev` resultaron tener exactamente el mismo blob SHA en `production`. Entre ellos:

- `packages/opencode/src/session/prompt.ts` → `0f85d44f209ba792065aeb951f0bd2e12b59fae8`
- `packages/opencode/src/session/processor.ts` → `20aa8a8404d8e5f50b0aafee5034ed6f1fa44382`
- `packages/opencode/src/session/llm.ts` → `a99f8acff20c5d64d0b6cb90df480218bb1daddc`
- `packages/opencode/src/session/compaction.ts` → `75d6374bfa54e5f492c2f0be83fa3029794009eb`
- `packages/opencode/src/agent/agent.ts` → `536a642fe49fb5211e66c2e2ad689856a03254c0`
- `packages/opencode/src/provider/provider.ts` → `b5980f15873b22647b03aa75fe450e2344aed5b9`
- `packages/opencode/src/provider/transform.ts` → `28a5beb9abacdf1546d9c3a4492b25e0e917f062`
- `packages/opencode/src/permission/index.ts` → `2e27ff2424dbb000ea9ed7f73471769716ba40a1`

Esta equivalencia permite reutilizar notas previas únicamente para esos blobs concretos. Si `production` avanza, debe repetirse la comparación.

## Re-audit trigger

Esta documentación deja de ser estrictamente representativa de producción cuando `production` deja de apuntar al commit de baseline. Para actualizarla debe compararse el nuevo commit contra `df35e842...`, priorizando cambios en:

- `packages/opencode/src/session/**`
- `packages/opencode/src/agent/**`
- `packages/opencode/src/tool/**`
- `packages/opencode/src/permission/**`
- `packages/opencode/src/provider/**`
- `packages/opencode/src/mcp/**`
- `packages/opencode/src/plugin/**`
- `packages/opencode/src/skill/**`
- `packages/core/src/session/**`
- `packages/llm/src/**`
- `packages/codemode/src/**`
- `packages/server/src/**`
- `packages/opencode/src/acp/**`
