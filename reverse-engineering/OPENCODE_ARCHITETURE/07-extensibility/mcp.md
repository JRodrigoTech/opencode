# MCP Integration

**Status:** VERIFIED-CODE

## MCP service responsibilities

`packages/opencode/src/mcp/index.ts` gestiona lifecycle de clientes MCP, conexiones, tools, instructions y capabilities. Archivos auxiliares cubren auth/OAuth, browser callback y catalog sanitization.

## Tool integration

Hay dos superficies principales:

### MCP tools

Las tools anunciadas por servidores conectados se convierten a tools consumibles por el modelo. Se someten a permissions y schema adaptation igual que el resto del capability plane.

### MCP resources

Si un client anuncia resources capability, `SessionTools.resolve` añade tools específicas para listar templates/resources y leer resources.

## Instructions integration

MCP servers pueden suministrar instrucciones. `SystemPrompt.mcp` filtra cada bloque según las tools asociadas y las permissions efectivas. Si todas las tools de un bloque están disabled, sus instrucciones no se inyectan.

Esto evita, al menos a nivel de policy, describir capacidades que el agent no puede utilizar.

## Resource attachment limits

Binary resource payloads:

- límite 10 MiB;
- attachments permitidos para PDF e imágenes GIF/JPEG/PNG/WEBP;
- texto se proyecta como contenido textual.

## OAuth/auth

El subsystem tiene módulos específicos `auth.ts`, `oauth-provider.ts`, `oauth-callback.ts` y browser helper. MCP no se reduce a stdio local; puede mantener flujos remotos autenticados.

## Code Mode relation

En Code Mode, las MCP tools pueden dejar de exponerse individualmente y ser accesibles a través del catálogo confinado de `execute`. Es una estrategia de orchestration distinta sobre la misma fuente de capabilities.

## Sources

- `packages/opencode/src/mcp/index.ts` — `05f12fa2ee4526e8b584fb367479cc55638cc7e6`
- `packages/opencode/src/session/tools.ts`
- `packages/opencode/src/session/system.ts`
- `packages/opencode/src/tool/code-mode.ts`
