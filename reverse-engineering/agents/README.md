# Work orders para agentes de reverse engineering

Este directorio contiene diez encargos independientes para analizar OpenCode por dominios arquitectónicos. Cada agente debe trabajar sobre las branches indicadas en su `PROMPT.md`, comparar siempre contra `dev` y escribir resultados exclusivamente en la branch `reverse-engineering` dentro de su directorio asignado bajo `reverse-engineering/analysis/`.

| Agente | Área | Prompt | Salida exclusiva |
|---|---|---|---|
| 01 | Arquitectura vigente | [`01-dev/PROMPT.md`](01-dev/PROMPT.md) | `analysis/01-dev/` |
| 02 | `beta` / `v2` / 2.0 | [`02-beta-v2/PROMPT.md`](02-beta-v2/PROMPT.md) | `analysis/02-beta-v2/` |
| 03 | Agents / subagents / skills | [`03-agent-subagent-skills/PROMPT.md`](03-agent-subagent-skills/PROMPT.md) | `analysis/03-agent-subagent-skills/` |
| 04 | Prompt / context / compaction | [`04-prompt-context-compaction/PROMPT.md`](04-prompt-context-compaction/PROMPT.md) | `analysis/04-prompt-context-compaction/` |
| 05 | Tools / permissions / code mode | [`05-tools-permissions-codemode/PROMPT.md`](05-tools-permissions-codemode/PROMPT.md) | `analysis/05-tools-permissions-codemode/` |
| 06 | Providers / LLM / Responses | [`06-providers-llm-responses/PROMPT.md`](06-providers-llm-responses/PROMPT.md) | `analysis/06-providers-llm-responses/` |
| 07 | Sessions / messages / events | [`07-session-message-events/PROMPT.md`](07-session-message-events/PROMPT.md) | `analysis/07-session-message-events/` |
| 08 | MCP / ACP | [`08-mcp-acp/PROMPT.md`](08-mcp-acp/PROMPT.md) | `analysis/08-mcp-acp/` |
| 09 | Backend / transports | [`09-backend-transports/PROMPT.md`](09-backend-transports/PROMPT.md) | `analysis/09-backend-transports/` |
| 10 | Effect / modularización | [`10-effect-modularization/PROMPT.md`](10-effect-modularization/PROMPT.md) | `analysis/10-effect-modularization/` |

## Contrato común

- Documentación en español; conservar términos técnicos ingleses cuando sean más precisos.
- No modificar código de producto ni documentación de otros agentes.
- Agrupar branches solo cuando la equivalencia/evolución esté demostrada mediante historia Git, commits, diffs o PRs.
- Incluir un `README.md` de síntesis en cada directorio de salida y subcarpetas por branch/familia según corresponda.
- Citar paths, símbolos, commits y PRs relevantes.
- Comparar contra `dev` y explicar qué cambió, por qué parece haberse creado la branch y qué terminó absorbido, descartado o divergente.
- Separar explícitamente evidencia confirmada, inferencias y preguntas abiertas.
