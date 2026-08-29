# Workspace Snapshots

**Status:** VERIFIED-CODE

## Integration point

`SessionProcessor` captura un snapshot antes de iniciar el LLM stream y otro al terminar cada step.

La captura inicial previa al stream es crítica porque una tool puede mutar el workspace antes de que el SDK emita `step-start`.

## Step diff

En `step-finish`:

1. se captura `nextSnapshot`;
2. `Snapshot.diff(previous, next)` produce hash/file list;
3. si hay archivos, `Snapshot.patch(hash)` genera patch textual;
4. se persiste `PatchPart` asociado al assistant message;
5. el snapshot actual avanza a `nextSnapshot`.

## Why patches are persisted

Patches sirven a varias capas:

- resumen de cambios de session;
- revert;
- audit trail del efecto de tools;
- UI/client visualization.

## Snapshot boundary

Snapshot representa estado del workspace observado por OpenCode, no el historial Git del proyecto. Puede usarse incluso aunque la carpeta no sea un repo Git.

## Sources

- `packages/opencode/src/snapshot/index.ts`
- `packages/opencode/src/session/processor.ts`
- `packages/opencode/src/session/revert.ts`
