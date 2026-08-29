# Server / HTTP API

**Status:** VERIFIED-CODE + DERIVED

## Authoritative contract

En production existe un package separado `packages/server`. `packages/client/README.md` indica que los clients se generan directamente desde la Effect `HttpApi` autoritativa de Server.

`packages/server/src` contiene:

- `api.ts`
- auth
- cors
- handlers
- location
- middleware
- routes.

## Host server bridge

`packages/opencode/src/server/server.ts` integra ese contract/handlers con el runtime de la aplicación, lifecycle global, authentication y routes específicas de instancia.

La antigua carpeta `opencode/src/server/routes` no debe interpretarse como el API completo: en la baseline contiene principalmente rutas instance-specific, mientras el contract estándar está extraído al package server/protocol.

## Event model

El host expone streaming/events de estado para que clientes observen session updates, permissions, tool progress, etc. EventV2Bridge es una pieza central que también alimenta plugins.

## Client generation invariant

El README del client declara tests de generation-equivalence para evitar drift entre:

- authoritative Effect HttpApi;
- Protocol projection;
- generated Effect client/runtime.

Custom transports como PTY WebSocket quedan fuera del cliente HTTP genérico.

## Security boundary

El server tiene auth/middleware separado del policy engine de tools. Server auth controla acceso externo al API; `Permission` controla acciones del agent dentro de una session. Son capas distintas y deben auditarse independientemente.

## Sources

- `packages/server/src/**`
- `packages/opencode/src/server/server.ts` — `440b992c...`
- `packages/client/README.md` — `8c53e47c7e681a43de56e307b355a8bc65d210a7`
