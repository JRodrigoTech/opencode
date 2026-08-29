# Source Map

Hotspots utilizados para el cruce. Los SHA son blob SHA cuando están fijados.

## Overmind baseline

| Área | Fuente | SHA / rol |
|---|---|---|
| Project direction | `README.md` | `d58948c241d46efd017bc2f04761013cd92d4c1a` |
| Current state | `AGENTIX/STATE.md` | `291a6d3a251a8b6372890f2653a87cc64e48e44d` |
| Runtime invariants | `AGENTIX/CURRENT_RUNTIME_BASELINE.md` | `8327a15ae395bb84b69a21ace1a8b72acff5e89b` |
| Agent loop | `overmind/agent.py` | `13ebf1d9d9e5c4042c21cc8be4170882ce5669b5` |
| Runtime readiness | `overmind/runtime.py` | `28c8a0d34ba6744d3211147d4bacee1bf1f1d131` |
| Composition root | `overmind/bootstrap.py` | `5d2d0feaff94315c677fd35c1817f77f7d677e68` |
| Context compiler | `overmind/context/compiler.py` | `3fa647344de49c47dd8f19d36ff0b0dd27a699aa` |
| Tool registry | `overmind/tools/registry.py` | `33fdf9c89b0d0ee9171b7601c6af51b29604fc7b` |
| Model service | `overmind/models/service.py` | `0579334ba92d91de31c87b8ad82c533fc11d885e` |
| Plugin composition | `overmind/plugins/composition.py` | `dc6a7fc8ac8d558b8e21b1b4cfd57d6c425f14d2` |
| Plugin target | `AGENTIX/PLUGIN_ARCHITECTURE/README.md` | `1938aabd33fcab256ad640d662c2104696a56240` |
| Plugin grants | `AGENTIX/PLUGIN_ARCHITECTURE/DEPENDENCY_AND_PERMISSION_CONTRACT.md` | `8db46cefbf8816dd1289c440f0f0d7bfbc29a469` |
| Events/services target | `AGENTIX/CORE_ARCHITECTURE/EVENTS_SERVICES_CONTRACT.md` | `84447872cbde0694598f4af56f3cd6413f969e85` |
| Subagent target | `AGENTIX/MODEL_ARCHITECTURE/MULTI_MODEL_AND_SUBAGENT_BOUNDARY.md` | `9fe5700688d062b6b2f6e6ad4171c4217059d69d` |
| Context refactor evidence | `AGENTIX/REFACTOR_CONTEXT_COMPILER/01_CURRENT_PROBLEMS.md` | `7a7a92a01c202a299dbe6eef039e1e04f259f0c4` |

## OpenCode baseline

| Área | Fuente | Blob SHA |
|---|---|---|
| Session loop | `packages/opencode/src/session/prompt.ts` | `0f85d44f209ba792065aeb951f0bd2e12b59fae8` |
| Event reducer | `packages/opencode/src/session/processor.ts` | `20aa8a8404d8e5f50b0aafee5034ed6f1fa44382` |
| LLM seam | `packages/opencode/src/session/llm.ts` | `a99f8acff20c5d64d0b6cb90df480218bb1daddc` |
| Native LLM adapter | `packages/opencode/src/session/llm/native-runtime.ts` | `bac385c59137ced710073051ed6388bc376e39ab` |
| Session tool adaptation | `packages/opencode/src/session/tools.ts` | `0f401c7562fa07076afd539990ca12fa207ceee0` |
| Tool registry | `packages/opencode/src/tool/registry.ts` | `9167cb3ea6bc5c8dd075f0f8271adbdec6074b12` |
| Subagent TaskTool | `packages/opencode/src/tool/task.ts` | `d8ca640cfba9a52d97e5180fda0ffa719910592b` |
| Agent policy | `packages/opencode/src/agent/agent.ts` | `536a642fe49fb5211e66c2e2ad689856a03254c0` |
| Permission engine | `packages/opencode/src/permission/index.ts` | `2e27ff2424dbb000ea9ed7f73471769716ba40a1` |
| Session | `packages/opencode/src/session/session.ts` | `a2a91cd47b5e854606444d1fc09fb18515fbe3b7` |
| Compaction | `packages/opencode/src/session/compaction.ts` | `75d6374bfa54e5f492c2f0be83fa3029794009eb` |
| Revert | `packages/opencode/src/session/revert.ts` | `03e5afd085e0181cf919de91360dd422b51e52bd` |
| Run state | `packages/opencode/src/session/run-state.ts` | `5cefdd04a3f3bb712c67c1f89687644a633509d0` |
| MCP | `packages/opencode/src/mcp/index.ts` | `05f12fa2ee4526e8b584fb367479cc55638cc7e6` |
| Plugin hooks | `packages/opencode/src/plugin/index.ts` | `6f05329a0833c0bb698572fba279e5bffc3bce49` |
| ACP | `packages/opencode/src/acp/service.ts` | `55fbc9681df3ba6d70364d47381d43979be87dbe` |

Para detalles OpenCode, usar `../OPENCODE_ARCHITETURE/SOURCE-INDEX.md` y `AUDIT-REPORT.md`.
