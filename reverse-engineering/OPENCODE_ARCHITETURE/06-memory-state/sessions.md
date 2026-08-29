# Sessions

**Status:** VERIFIED-CODE

## Persistent identity

`Session.Info` contiene:

- `id`, `slug`
- `projectID`, opcional `workspaceID`
- `directory`, opcional `path`
- opcional `parentID`
- `title`
- opcional `agent`
- opcional `model { id, providerID, variant }`
- `version`
- summary de cambios
- cost/tokens
- share URL
- metadata
- time created/updated/compacting/archived
- opcional session permission ruleset
- opcional revert state.

## Storage mapping

`fromRow` y `toRow` mapean `SessionTable` a `Session.Info`. La tabla separa columnas de token usage:

- input
- output
- reasoning
- cache read
- cache write.

El coste se almacena acumulado a nivel session además del usage por step/message.

## Parent/child sessions

`parentID` forma parte del schema persistente, no metadata ad hoc. Se usa para subagents/background work y permite modelar jerarquía real de ejecución.

## Default titles

Hay títulos sintéticos con prefijos:

- `New session - <ISO>`
- `Child session - <ISO>`

`isDefaultTitle` reconoce ambos. Los forks incrementan sufijos `(fork #N)`.

## Revert state on session

El estado revert persistido incluye:

- `messageID`
- opcional `partID`
- opcional snapshot
- opcional diff.

Así, revert es parte del estado transaccional de la session y no una operación puramente filesystem.

## Accounting

El runtime calcula usage/cost por response y lo agrega a session. La lógica trata metadata de providers distintos para normalizar cache/reasoning y pricing.

## Sources

- `packages/opencode/src/session/session.ts` — `a2a91cd47b5e854606444d1fc09fb18515fbe3b7`
- `packages/core/src/session/sql/**`
