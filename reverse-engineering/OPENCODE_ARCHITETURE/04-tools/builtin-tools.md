# Built-in Tools Inventory

**Status:** VERIFIED-CODE at registry level

| Tool | Principal role | Notas de policy/runtime |
|---|---|---|
| `invalid` | Capturar tool calls inválidas/no reparables | Fallback del LLM adapter |
| `shell` | Ejecutar comandos | External-directory/command policy relevante |
| `read` | Leer archivos/directorios | Puede cargar nested instructions |
| `glob` | Buscar por patrón de paths | Read-only |
| `grep` | Buscar contenido | Usa infraestructura ripgrep/core |
| `edit` | Edición textual | Se oculta para ciertos GPT donde se usa patch |
| `write` | Escritura de archivo | Mismo gating que edit |
| `apply_patch` | Aplicar patches | Preferido por algunos GPT |
| `task` | Delegar a subagent | Description dinámica y task permissions |
| `todowrite` | Manipular todo state | Denegado en algunos subagents |
| `webfetch` | Obtener contenido web | Tool host |
| `websearch` | Búsqueda web | Gated por provider/flags |
| `skill` | Cargar skill | System prompt lista skills si permitido |
| `question` | Pedir input al cliente | Solo ciertos clients/flag |
| `lsp` | Operaciones Language Server | Experimental flag |
| `plan_exit` | Transición de plan mode | Experimental + CLI |
| `execute` | Orquestación confinada de MCP | Code Mode experimental |

## MCP resource pseudo-tools

`SessionTools` puede añadir además:

- `list_mcp_resources`
- `list_mcp_resource_templates`
- `read_mcp_resource`

No proceden del ToolRegistry built-in normal; se construyen al resolver sesión cuando existen MCP clients con capability de resources.

## Attachment policy for MCP resources

`read_mcp_resource` admite attachments binarios solo para tipos soportados y limita blobs a 10 MiB. MIME soportados observados:

- PDF
- GIF
- JPEG
- PNG
- WEBP

Los contenidos text se incorporan como texto.

## Source

- `packages/opencode/src/tool/registry.ts`
- `packages/opencode/src/session/tools.ts`
