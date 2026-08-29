# Code Mode

**Status:** VERIFIED-CODE

## Concept

Code Mode introduce una tool host llamada `execute` cuya descripción de baseline es conceptualmente “Run a confined orchestration script with access to connected MCP tools”.

En lugar de exponer muchas tools MCP como llamadas LLM separadas, permite al modelo escribir un pequeño script que invoca varias dentro de un interpreter confinado.

## Flow

```text
LLM
 │
 └─ tool: execute(code)
        │
        ▼
@opencode-ai/codemode interpreter
        │
        ├─ MCP tool A
        ├─ MCP tool B
        └─ MCP tool C
        │
        ▼
combined result + child-call metadata
```

## Host bridge

`opencode/src/tool/code-mode.ts` construye el catálogo namespaced desde MCP, aplica permissions y hooks a child calls, recoge attachments/logs/metadata y soporta abort.

## Registry gating

`ToolRegistry` solo inicializa Code Mode si `experimentalCodeMode` está activo. Además elimina `execute` si después de permissions no existe catálogo MCP visible útil.

## Package runtime

`packages/codemode` contiene:

- `codemode.ts`
- interpreter
- OpenAPI helpers
- stdlib
- `tool-runtime.ts`
- tool schemas/values.

La separación permite que el interpreter no dependa del agent loop completo.

## Performance/behavior implication

Sin Code Mode:

`LLM → MCP A → LLM → MCP B → LLM ...`

Con Code Mode:

`LLM → execute(script) → A/B/C → combined result`

Puede reducir round trips del modelo y mantener control flow determinista dentro del script, a cambio de añadir un sandbox/orchestration boundary que debe auditarse separadamente.

## Sources

- `packages/opencode/src/tool/code-mode.ts` — `332d4b43f150dcf9047ac70bb5313ecd4e187121`
- `packages/codemode/src/tool-runtime.ts` — `f4ccc61d4c49a4f7572906559e5a4e2a11acdec9`
- `packages/codemode/src/codemode.ts` — `14782d7c...`
