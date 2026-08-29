# OpenCode Production Architecture — Reverse Engineering

> **Source of truth:** `production@df35e842f59bc115bb7c0479a8e11f017d443f2c`  
> **Documentation branch:** `reverse-engineering`  
> **Repository:** `JRodrigoTech/opencode`  
> **Audit date:** 2026-08-29

Este árbol documenta mediante ingeniería inversa la arquitectura efectiva de OpenCode en la rama `production`. El objetivo no es repetir la documentación pública, sino reconstruir el comportamiento del sistema desde el código: composición del contexto, state machine del agente, tool runtime, permisos, providers, persistencia, compaction, snapshots, extensibilidad y superficies externas.

## Regla de auditoría

Cada documento separa tres categorías:

- **VERIFIED-CODE**: afirmación observada directamente en código de la baseline fijada.
- **DERIVED**: conclusión arquitectónica deducida de varias piezas de código, con las fuentes indicadas.
- **OPEN**: punto que requiere una auditoría adicional o ejecución controlada; no se presenta como hecho.

Cuando un fichero crítico coincide byte a byte con el ya estudiado en otra rama, se acepta reutilizar el análisis únicamente si el **blob SHA de Git es idéntico**. La baseline de esta documentación siempre es `production`, no `dev`.

## Mapa de documentación

```text
OPENCODE_ARCHITETURE/
├── README.md
├── AUDIT-BASELINE.md
├── SOURCE-INDEX.md
├── 00-architecture/
│   ├── overview.md
│   ├── packages-and-boundaries.md
│   └── dependency-graph.md
├── 01-agent-runtime/
│   ├── agent-loop.md
│   ├── processor-state-machine.md
│   ├── run-state-and-cancellation.md
│   └── errors-retries.md
├── 02-context-engine/
│   ├── system-prompt.md
│   ├── model-prompts.md
│   ├── instructions.md
│   ├── message-model.md
│   └── context-assembly.md
├── 03-agents/
│   ├── agents.md
│   ├── subagents.md
│   ├── task-tool.md
│   └── permissions-by-agent.md
├── 04-tools/
│   ├── tool-runtime.md
│   ├── registry.md
│   ├── builtin-tools.md
│   ├── tool-lifecycle.md
│   └── permission-engine.md
├── 05-llm/
│   ├── llm-abstraction.md
│   ├── ai-sdk-runtime.md
│   ├── native-runtime.md
│   ├── provider-transform.md
│   └── provider-matrix.md
├── 06-memory-state/
│   ├── sessions.md
│   ├── messages-and-parts.md
│   ├── compaction.md
│   ├── snapshots.md
│   └── revert.md
├── 07-extensibility/
│   ├── mcp.md
│   ├── plugins.md
│   ├── skills.md
│   └── code-mode.md
└── 08-interfaces/
    ├── server-api.md
    ├── acp.md
    ├── cli.md
    └── clients.md
```

## Lectura recomendada

Para entender el agente, el orden más eficiente es:

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

OpenCode production es un **agent runtime dirigido por eventos** construido sobre Effect. `SessionPrompt` orquesta turnos y ciclos; `SessionProcessor` reduce un stream normalizado de `LLMEvent` a mensajes y partes persistentes; `SessionTools` y `ToolRegistry` producen el conjunto de capacidades visible para el modelo; `Permission` aplica políticas; `LLM` abstrae el transporte y normaliza dos runtimes posibles; `SessionCompaction`, `Snapshot` y `SessionRevert` proporcionan continuidad y reversibilidad.

La unidad fundamental no es “un prompt”, sino una **Session** cuyo estado se materializa como mensajes y parts, y cuya ejecución puede producir múltiples steps y tool calls antes de volver a `idle`.
