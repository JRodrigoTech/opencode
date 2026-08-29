# Instruction Discovery and Injection

**Status:** VERIFIED-CODE

`Instruction.Service` gestiona instrucciones persistentes del entorno/proyecto y reglas locales descubiertas durante reads.

## Global candidates

Se consideran:

- `${global.config}/AGENTS.md`
- `${home}/.claude/CLAUDE.md` salvo flag `disableClaudeCodePrompt`

Para este conjunto global se usa el primer candidato existente.

## Project candidates

Nombres:

1. `AGENTS.md`
2. `CLAUDE.md` salvo flag de disable
3. `CONTEXT.md` (deprecated)

Si project config está habilitada, se busca hacia arriba desde current directory hasta worktree. La primera **familia de filename** con matches gana; no se apilan automáticamente AGENTS y CLAUDE de todos los ancestros.

## Config instructions

`config.instructions` puede contener:

- paths/globs locales;
- `~/...`;
- URLs `http://` o `https://`.

Las URLs se descargan con timeout y retry HTTP transitorio; fallo produce contenido vacío en vez de romper el loop.

## System injection

`system()` produce entries `Instructions from: <source>\n<content>` para paths y URLs resueltos.

## Local nested instructions during reads

`resolve(messages, filepath, messageID)` hace una segunda forma de context injection. Desde el directorio del fichero leído camina hacia arriba dentro del root y busca instruction files cercanos.

Evita duplicados mediante:

- paths de instrucciones de system;
- paths ya cargados por tool `read` en mensajes previos (`metadata.loaded`);
- `claims` por assistant message.

La consecuencia es importante: **leer un fichero puede ampliar el contexto de instrucciones del agente** según la jerarquía de directorios.

## Claim lifetime

Los claims son por `messageID` y se borran con `Instruction.clear(messageID)`, llamado por `SessionPrompt` al finalizar la iteración correspondiente.

## Sources

- `packages/opencode/src/session/instruction.ts` — `7f593550d468fa3ae5dbc6c04ce53f317bb72533`
