# Skills

**Status:** VERIFIED-CODE

## Separation from agents/plugins

Una skill es un paquete de instrucciones/workflow descubierto y cargado bajo demanda mediante `skill` tool. No crea necesariamente una child session y no es equivalente a plugin code.

## Discovery

El subsystem está dividido en:

- `skill/discovery.ts`
- `skill/index.ts`

El catálogo agrega skills disponibles desde locations/config sources y expone metadata para matching/visualización.

## System prompt exposure

`SystemPrompt.skills(agent)`:

1. comprueba si permission `skill` está disabled;
2. obtiene `skill.available(agent)`;
3. inyecta explicación + catálogo verbose en system context.

El modelo conoce así nombre/description antes de llamar a la skill tool.

## Tool execution

`SkillTool` carga las instrucciones de una skill concreta. El output puede quedar en el historial como tool result; compaction pruning protege los outputs de skill frente al pruning normal de tool outputs antiguos.

## Permission boundary

La disponibilidad de una skill puede depender del agent y la tool puede ocultarse globalmente por permissions. El system context respeta ese mismo boundary.

## Source

- `packages/opencode/src/skill/index.ts` — `5a04ec213994a65dd25098b843efca1fbd1c4e0e`
- `packages/opencode/src/skill/discovery.ts` — `b56a6761...`
- `packages/opencode/src/tool/skill.ts`
- `packages/opencode/src/session/system.ts`
