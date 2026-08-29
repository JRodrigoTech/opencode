# Source Index

**Baseline:** `production@df35e842f59bc115bb7c0479a8e11f017d443f2c`

Índice de hotspots usados para reconstruir la arquitectura. Los SHA son blob SHA de Git, no commit SHA.

| Área | Fuente | Blob SHA | Rol |
|---|---|---|---|
| Loop | `packages/opencode/src/session/prompt.ts` | `0f85d44f209ba792065aeb951f0bd2e12b59fae8` | Orquestación de prompt, loop, shell y command |
| Processor | `packages/opencode/src/session/processor.ts` | `20aa8a8404d8e5f50b0aafee5034ed6f1fa44382` | Reductor de `LLMEvent`, tool state, snapshots, usage |
| LLM seam | `packages/opencode/src/session/llm.ts` | `a99f8acff20c5d64d0b6cb90df480218bb1daddc` | Selección AI SDK/native y stream normalizado |
| Native LLM adapter | `packages/opencode/src/session/llm/native-runtime.ts` | `bac385c59137ced710073051ed6388bc376e39ab` | Gating/adaptación del runtime `@opencode-ai/llm` |
| Tool resolve | `packages/opencode/src/session/tools.ts` | `0f401c7562fa07076afd539990ca12fa207ceee0` | Adapta registry/MCP a tools del modelo |
| Tool registry | `packages/opencode/src/tool/registry.ts` | `9167cb3ea6bc5c8dd075f0f8271adbdec6074b12` | Builtins/custom/plugins/model gating |
| Task/subagents | `packages/opencode/src/tool/task.ts` | `d8ca640cfba9a52d97e5180fda0ffa719910592b` | Child sessions, resume, background jobs y delegación |
| Agents | `packages/opencode/src/agent/agent.ts` | `536a642fe49fb5211e66c2e2ad689856a03254c0` | Definición y configuración de agentes |
| System context | `packages/opencode/src/session/system.ts` | `d0c608b203f68f8c84f117129852b30c9b73d090` | Environment, skills, MCP instructions |
| Instructions | `packages/opencode/src/session/instruction.ts` | `7f593550d468fa3ae5dbc6c04ce53f317bb72533` | AGENTS/CLAUDE/CONTEXT/config instruction discovery |
| Message model | `packages/opencode/src/session/message-v2.ts` | `9b3f2c46f40578128001957004c67633a18da23a` | Parts y conversión a mensajes de modelo |
| Session | `packages/opencode/src/session/session.ts` | `a2a91cd47b5e854606444d1fc09fb18515fbe3b7` | Modelo persistente y operaciones de sesión |
| Compaction | `packages/opencode/src/session/compaction.ts` | `75d6374bfa54e5f492c2f0be83fa3029794009eb` | Resumen, tail preservation y pruning |
| Retry | `packages/opencode/src/session/retry.ts` | `284c0f0ade4143df6aae127e60c511f5466c46a6` | Política de retry |
| Revert | `packages/opencode/src/session/revert.ts` | `03e5afd085e0181cf919de91360dd422b51e52bd` | Reversión de workspace + historia |
| Run state | `packages/opencode/src/session/run-state.ts` | `5cefdd04a3f3bb712c67c1f89687644a633509d0` | Runner por sesión y cancelación |
| Permission | `packages/opencode/src/permission/index.ts` | `2e27ff2424dbb000ea9ed7f73471769716ba40a1` | Evaluación, ask/reply, approvals |
| Provider registry | `packages/opencode/src/provider/provider.ts` | `b5980f15873b22647b03aa75fe450e2344aed5b9` | Providers/model loading |
| Provider transform | `packages/opencode/src/provider/transform.ts` | `28a5beb9abacdf1546d9c3a4492b25e0e917f062` | Normalización provider/model-specific |
| MCP | `packages/opencode/src/mcp/index.ts` | `05f12fa2ee4526e8b584fb367479cc55638cc7e6` | MCP clients/tools/resources/instructions |
| Plugin core | `packages/opencode/src/plugin/index.ts` | `6f05329a0833c0bb698572fba279e5bffc3bce49` | Carga y ejecución ordenada de hooks |
| Skills | `packages/opencode/src/skill/index.ts` | `5a04ec213994a65dd25098b843efca1fbd1c4e0e` | Catálogo/disponibilidad de skills |
| Code Mode bridge | `packages/opencode/src/tool/code-mode.ts` | `332d4b43f150dcf9047ac70bb5313ecd4e187121` | Tool `execute` sobre MCP |
| LLM package | `packages/llm/src/llm.ts` | `e4781d8608b0185c500866aae20fda8335640550` | Contratos/eventos del runtime nativo |
| LLM tool runtime | `packages/llm/src/tool-runtime.ts` | `d69bbb9d478ca532a1481e8d7502c8e5c2b55dc6` | Dispatch de tools del runtime LLM |
| Code Mode runtime | `packages/codemode/src/tool-runtime.ts` | `f4ccc61d4c49a4f7572906559e5a4e2a11acdec9` | Ejecución de catálogo confinado |
| ACP | `packages/opencode/src/acp/service.ts` | `55fbc9681df3ba6d70364d47381d43979be87dbe` | Adaptador Agent Client Protocol |
| HTTP contract | `packages/server/src/api.ts` | `981ad28db93d253ac02e231b6dbc28f034fe5c35` | API Effect autoritativa |
| Generated clients | `packages/client/README.md` | `8c53e47c7e681a43de56e307b355a8bc65d210a7` | Boundary Promise/Effect derivado de HttpApi |

## Cómo usar este índice

Para auditar una afirmación, localizar primero el documento temático y después seguir sus `Sources`. El fichero de mayor tamaño no es necesariamente el de mayor autoridad: `SessionPrompt` decide el ciclo, `SessionProcessor` materializa el stream, `TaskTool` posee la delegación y `Permission` decide autorización.

La segunda auditoría está registrada en `AUDIT-REPORT.md`.
