# Revert / Unrevert

**Status:** VERIFIED-CODE

## Preconditions

`SessionRevert.revert` llama `runState.assertNotBusy(sessionID)`. No se permite revert mientras el runner de la session está activo.

## Revert algorithm

El runtime recorre messages/parts desde el boundary solicitado y recopila patches posteriores. Captura snapshot actual, aplica reversión de patches/workspace, calcula diff y actualiza summary/revert state de la session.

El `revert` state conserva suficiente información para posibilitar `unrevert`.

## History cleanup is deferred

Una característica importante: revert del workspace no elimina inmediatamente toda la historia posterior. `cleanup(session)` se ejecuta al inicio del siguiente prompt y elimina messages/parts después del boundary revert antes de continuar.

Esto permite conservar la posibilidad de unrevert entre la acción revert y la siguiente edición conversacional.

## Unrevert

`unrevert` restaura el snapshot guardado y limpia el estado revert. También requiere consistencia con runner/session state.

## Transactional interpretation

Revert debe entenderse como una operación coordinada sobre:

1. workspace state;
2. session metadata;
3. execution history boundary.

No es equivalente a `git reset`, ni depende necesariamente de Git.

## Sources

- `packages/opencode/src/session/revert.ts` — `03e5afd085e0181cf919de91360dd422b51e52bd`
- `packages/opencode/src/snapshot/index.ts`
- `packages/opencode/src/session/session.ts`
