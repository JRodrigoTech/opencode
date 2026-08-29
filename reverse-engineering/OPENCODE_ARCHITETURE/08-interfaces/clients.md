# Generated and Application Clients

**Status:** VERIFIED-CODE

## `@opencode-ai/client`

El README de production define dos entrypoints:

- `@opencode-ai/client`: Promise client sin Effect runtime, basado en `fetch`.
- `@opencode-ai/client/effect`: cliente Effect rico usando un `HttpClient` del environment.

## Generation source

La superficie generada incluye todos los grupos HTTP estándar del Server concrete API. El compilador lee `@opencode-ai/server/api`.

El Effect runtime generado usa una proyección local derivada de Protocol y existen tests de equivalencia para evitar transport drift.

## Type boundary

El entrypoint Effect usa valores decodificados canónicos como:

- `Session.ID`
- `Location.Ref`
- `Prompt`.

Estos datatypes provienen del package ligero `@opencode-ai/schema` y se re-exportan desde client.

## Browser/runtime boundary

Según el README:

- Promise root no depende de Core/Effect runtime;
- `/effect` depende de Effect, Schema y Protocol;
- tests de bundle boundary protegen esos graphs.

## Custom transports

PTY WebSocket no se genera como parte del generic HTTP client. Esto confirma que no toda la interacción con OpenCode utiliza el mismo transporte.

## Other clients

El monorepo incluye surfaces adicionales (`app`, `desktop`, etc.). Para reverse engineering del agente deben tratarse como consumers del mismo session/event/capability model salvo que una flag de client altere tool exposure o UI-mediated permissions.

## Source

- `packages/client/README.md` — `8c53e47c7e681a43de56e307b355a8bc65d210a7`
- `packages/server/src/api.ts`
- `packages/opencode/src/tool/registry.ts`
