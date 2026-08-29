# Packages and Boundaries

**Status:** VERIFIED-CODE + DERIVED

## Runtime split

El agente no vive en un solo package. La baseline production contiene varios boundaries relevantes:

| Package | Boundary observado | Qué no debe confundirse con él |
|---|---|---|
| `packages/opencode` | Host runtime: sessions, agents, tools, providers, MCP, plugins, ACP glue | No es el único dueño de contratos ni transporte LLM |
| `packages/core` | Servicios y datatypes compartidos: database, session contracts/SQL, Effect helpers, project/workspace, FS/process utilities | No contiene por sí solo el agent loop |
| `packages/llm` | Runtime/protocol abstraction nativa para LLM, events, providers, route y tool runtime | No decide la política de agents/tools de OpenCode |
| `packages/codemode` | Interpreter/tool orchestration confinado y schemas de Code Mode | La integración con sesiones/MCP está en `opencode/src/tool/code-mode.ts` |
| `packages/server` | Contrato HTTP Effect y middleware/handlers server-side | No es la antigua carpeta `opencode/src/server` únicamente |
| `packages/client` | Clientes generados desde la HttpApi autoritativa | No contiene lógica del agente |
| `packages/cli` | Superficie CLI/TUI y command framework | Debe tratarse como cliente/host surface |

## Evidence from `packages/opencode/package.json`

El package `opencode` en production versión `1.18.25` depende explícitamente de workspaces:

- `@opencode-ai/core`
- `@opencode-ai/codemode`
- `@opencode-ai/llm`
- `@opencode-ai/plugin`
- `@opencode-ai/protocol`
- `@opencode-ai/schema`
- `@opencode-ai/sdk`
- `@opencode-ai/server`
- `@opencode-ai/tui`

Esto confirma que el runtime host integra módulos separados en vez de encapsularlos internamente.

## Ownership model

### Host/policy ownership: `packages/opencode`

Ejemplos:

- selección de agent: `src/agent/agent.ts`
- tool set: `src/tool/registry.ts`
- authorization: `src/permission/index.ts`
- session loop: `src/session/prompt.ts`
- tool adaptation: `src/session/tools.ts`
- provider registry/transform: `src/provider/*`

### Contract/infrastructure ownership: `packages/core`

El código host importa de Core, entre otros:

- `@opencode-ai/core/v1/session`
- `@opencode-ai/core/session/sql`
- `@opencode-ai/core/database/database`
- `@opencode-ai/core/provider`
- `@opencode-ai/core/model`
- `@opencode-ai/core/effect/layer-node`

Por tanto, los schemas persistentes y primitives de runtime atraviesan el boundary del package.

### LLM transport ownership split

`packages/opencode/src/session/llm.ts` conserva la política de preparación y selección de runtime. El runtime nativo delega transporte/routing a `@opencode-ai/llm`; el camino AI SDK usa `streamText`. OpenCode mantiene la ejecución de tools y adapta ambos caminos al mismo event stream.

### HTTP contract ownership

`packages/client/README.md` declara que los clientes se generan directamente desde la Effect `HttpApi` autoritativa de Server y que hay tests de equivalencia para prevenir drift. Esto es importante: el cliente no debe tomarse como definición primaria del API.

## Reverse-engineering consequence

Un call graph completo debe atravesar packages. Limitar el estudio a `packages/opencode` ocultaría:

- contratos de sesión/SQL;
- event model nativo de LLM;
- routing/transport;
- Code Mode interpreter;
- HttpApi autoritativa;
- clientes derivados.

## Sources

- `packages/opencode/package.json` — `45c61103...`
- `packages/core/src/**`
- `packages/llm/src/**`
- `packages/codemode/src/**`
- `packages/server/src/**`
- `packages/client/README.md` — `8c53e47c...`
