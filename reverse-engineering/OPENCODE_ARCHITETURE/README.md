# OpenCode Production Architecture — Reverse Engineering

> **Source of truth:** `production@df35e842f59bc115bb7c0479a8e11f017d443f2c`  
> **Documentation branch:** `reverse-engineering`  
> **Repository:** `JRodrigoTech/opencode`  
> **Audit date:** 2026-08-29

Este árbol documenta mediante ingeniería inversa la arquitectura efectiva de OpenCode en la rama `production`. El objetivo es reconstruir comportamiento desde código: contexto, state machine del agente, tools, permisos, providers, persistencia, compaction, snapshots, extensibilidad y superficies externas.

## Audit status

La documentación fue sometida a una **segunda auditoría estática** contra el commit congelado de production. El resultado, correcciones de trazabilidad y límites de validación están en [`AUDIT-REPORT.md`](AUDIT-REPORT.md).

Categorías usadas:

- **VERIFIED-CODE**: observado directamente en código de la baseline.
- **DERIVED**: conclusión arquitectónica cruzada entre varias fuentes.
- **OPEN**: requiere análisis adicional o validación dinámica.

## Mapa

```text
OPENCODE_ARCHITETURE/
├── README.md
├── AUDIT-BASELINE.md
├── AUDIT-REPORT.md
├── SOURCE-INDEX.md
├── 00-architecture/       # boundaries y dependency graph
├── 01-agent-runtime/      # loop, processor, cancellation, retry
├── 02-context-engine/     # prompt/context/instructions/messages
├── 03-agents/             # agents, subagents, TaskTool, policy
├── 04-tools/              # registry, lifecycle, permission engine
├── 05-llm/                # AI SDK/native/provider adaptation
├── 06-memory-state/       # sessions, compaction, snapshot/revert
├── 07-extensibility/      # MCP, plugins, skills, Code Mode
└── 08-interfaces/         # HTTP/server, ACP, CLI, clients
```

## Lectura recomendada

1. `00-architecture/overview.md`
2. `01-agent-runtime/agent-loop.md`
3. `01-agent-runtime/processor-state-machine.md`
4. `02-context-engine/context-assembly.md`
5. `04-tools/tool-runtime.md`
6. `03-agents/subagents.md`
7. `05-llm/llm-abstraction.md`
8. `06-memory-state/compaction.md`
9. `07-extensibility/*`
10. `08-interfaces/*`

## Tesis arquitectónica

OpenCode production es un **agent runtime stateful dirigido por eventos** sobre Effect. `SessionPrompt` orquesta; `SessionProcessor` reduce `LLMEvent` a estado persistente; `SessionTools` y `ToolRegistry` construyen capacidades; `Permission` aplica policy; `LLM` normaliza AI SDK/native; `SessionCompaction`, `Snapshot` y `SessionRevert` aportan continuidad y reversibilidad.

La unidad fundamental no es un prompt aislado sino una **Session** cuyos messages/parts registran diálogo, reasoning, tools, steps, patches y errores, y cuya ejecución puede abarcar múltiples model turns antes de volver a `idle`.
