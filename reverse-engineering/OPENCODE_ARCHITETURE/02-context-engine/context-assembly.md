# Context Assembly — End-to-End

**Status:** VERIFIED-CODE + DERIVED

Este documento resume qué información llega conceptualmente a una llamada del modelo y en qué fase se incorpora.

## Pipeline

```text
Persisted session history
    │
    ├─ filter compacted history
    ├─ latest-state analysis
    ├─ reminders
    └─ plugin message transforms
    │
    ▼
MessageV2.toModelMessagesEffect
    │
    ▼
ModelMessage[]

System side:
    model/provider base prompt (during LLM request preparation)
    + environment
    + global/project/config instructions
    + MCP instructions
    + skills catalog
    + structured-output directive when applicable
    + agent/provider request transforms
```

## Phase 1 — persisted history

El loop carga history compactado. Esto puede excluir/representar de forma resumida historia antigua. Antes de llamar al LLM se aplican reminders y el hook `experimental.chat.messages.transform`.

## Phase 2 — system components

En paralelo se resuelven:

- `sys.skills(agent)`
- `sys.environment(model)`
- `instruction.system()`
- `sys.mcp(agent, session.permission)`
- model messages.

El array lógico se ordena: environment → instructions → MCP → skills.

## Phase 3 — agent/model/provider preparation

`LLMRequestPrep.prepare` recibe agent, provider, auth, tools, flags y mensajes. Es aquí donde el model-specific base prompt y provider options pueden reestructurar la solicitud final. `ProviderTransform.message` vuelve a adaptar el prompt para el modelo en el AI SDK middleware y en el runtime nativo.

## Phase 4 — last-step pressure

Cuando el loop alcanza el límite de steps, añade un assistant content `MAX_STEPS_PROMPT` al final de messages. Es una intervención de control del runtime, no un mensaje histórico de usuario.

## File/resource expansion

Antes de entrar al loop, `createUserMessage` puede expandir inputs:

- `file:` text → ejecución de Read y text sintético;
- directory → Read directory;
- binary local → data URL file part;
- MCP resource → text o attachments permitidos;
- agent part → instrucción sintética para task.

Por tanto, dos inputs visualmente similares pueden producir context payloads distintos según MIME/protocolo/source.

## Key invariant

El “prompt efectivo” debe auditarse como resultado de un pipeline. Guardar solo los `.txt` de prompts no reconstruye el contexto real.

## Sources

- `session/prompt.ts`
- `session/system.ts`
- `session/instruction.ts`
- `session/message-v2.ts`
- `session/llm.ts`
- `provider/transform.ts`
