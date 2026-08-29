# ACP — Agent Client Protocol

**Status:** VERIFIED-CODE

## Surface

`packages/opencode/src/acp` implementa el adaptador de Agent Client Protocol y contiene módulos para:

- agent
- config options
- content conversion
- directory
- errors
- events
- permissions
- profiles
- service
- session
- tool
- usage.

## Core service

`acp/service.ts` es el mayor coordinator de esta superficie. Traduce llamadas/eventos ACP al modelo interno de OpenCode y viceversa.

## Mapping boundary

ACP no reemplaza el agent runtime. Es una interfaz externa que debe mapear:

- sessions ACP ↔ OpenCode sessions;
- content blocks ↔ messages/parts;
- tool calls ↔ tool state;
- permission requests ↔ PermissionV1;
- usage ↔ normalized token/cost data.

## Why separate adapters exist

Protocol semantics no coinciden necesariamente 1:1 con OpenCode. Por eso hay ficheros dedicados a `content.ts`, `tool.ts`, `permission.ts`, `usage.ts`: la conversión es explícita y auditable.

## RE implication

Cuando un comportamiento solo ocurre mediante un editor/cliente ACP, primero debe determinarse si el core runtime produjo el mismo state y el adapter lo transformó distinto. No debe atribuirse automáticamente al agent loop.

## Sources

- `packages/opencode/src/acp/service.ts` — `55fbc9681df3ba6d70364d47381d43979be87dbe`
- `packages/opencode/src/acp/event.ts` — `dec34b8b...`
- `packages/opencode/src/acp/tool.ts` — `309c126d...`
- `packages/opencode/src/acp/permission.ts` — `4eeca28f...`
