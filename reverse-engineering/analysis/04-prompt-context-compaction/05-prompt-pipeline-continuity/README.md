# Prompt pipeline, metadata y continuidad entre turns

## Corrección de auditoría

No existe un único pipeline de prompt en `dev`. Este documento debe separar:

- **pipeline de producto**: `packages/opencode/src/session/prompt.ts` (`SessionPrompt`);
- **pipeline V2/Core**: `packages/core/src/session/runner/llm.ts` (`SessionRunner`).

La versión anterior presentaba el segundo como si hubiese reemplazado al primero. El composition root de `packages/opencode` demuestra que `SessionPrompt` sigue siendo un servicio real y cableado.

---

## 1. Pipeline de producto: `SessionPrompt`

### Admisión y persistencia del user turn

`prompt(input)`:

1. carga la Session;
2. limpia revert si procede;
3. crea y persiste el user message mediante `createUserMessage()`;
4. actualiza/toca la Session;
5. materializa overrides de tools como reglas de permiso cuando existen;
6. si `noReply` es false entra en `loop({ sessionID })`.

`createUserMessage()` resuelve:

- agente explícito o default;
- modelo explícito, modelo del agente o modelo actual de la Session;
- variant válida;
- parts text/file/agent/subtask;
- attachments y recursos MCP;
- hooks de plugin;
- agent/model durables de la Session.

### Loop de ejecución

`runLoop()` recarga historia mediante `MessageV2.filterCompactedEffect()` en cada iteración y decide si:

- el assistant anterior ya terminó;
- hay una subtask pendiente;
- debe ejecutar otro provider turn;
- debe compactar;
- debe continuar tras tools.

En cada provider turn obtiene:

```text
skills       = SystemPrompt.skills(agent)
environment  = SystemPrompt.environment(model)
instructions = Instruction.system()
mcp          = SystemPrompt.mcp(agent, session.permission)
messages     = MessageV2.toModelMessagesEffect(history, model)
```

Luego construye `system[]`, añade structured-output guidance cuando corresponde y llama a `SessionProcessor.process(...)` con modelo, tools, permisos, system e historial proyectado.

### Autoridad y continuidad en este path

La continuidad se basa en:

- filas durable/proyectadas de Session, Message y Part;
- `MessageV2.filterCompactedEffect()`;
- compaction legacy-compatible;
- re-resolución de agente/modelo/tools por iteración;
- reconstrucción de `system[]` desde entorno/instrucciones/skills/MCP;
- provider metadata y transforms preservados por las capas `MessageV2`/`LLM` cuando son aplicables.

No existe en este path una dependencia obligatoria de `SessionContextEpoch` para recordar qué system sources conoce el modelo.

### Cambio de agente/modelo

`createUserMessage()` actualiza `Session.agent`/`Session.model` si cambian. El siguiente loop utiliza esa nueva selección para tools, system y modelo. La continuidad es por estado durable de la Session y el transcript, no por conservar una request completa.

### Headers y metadata de transporte

El path de producto delega la preparación final a `packages/opencode/src/session/llm.ts` y `session/llm/request.ts`, donde se añaden opciones/headers/provider transforms según modelo, integración y runtime elegido. No debe copiarse automáticamente la lista exacta de headers del runner V2 a este path.

---

## 2. Pipeline V2/Core: `SessionRunner`

Archivo: `packages/core/src/session/runner/llm.ts`.

Para cada attempt V2:

1. carga Session/location;
2. selecciona agente;
3. combina `SystemContext`, skill guidance y reference guidance;
4. inicializa/prepara `SessionContextEpoch`;
5. resuelve modelo;
6. carga `SessionHistory.entriesForRunner()` desde `baselineSeq`;
7. materializa tools;
8. construye `LLM.request()`;
9. puede compactar antes del provider call;
10. ejecuta `llm.stream(request)`;
11. persiste eventos assistant/tool/usage;
12. decide continuación, steering o compaction.

### `request.system` V2

Se forma a partir de:

```text
agent.info?.system
system.baseline
```

Los updates posteriores al baseline viajan en la historia V2.

### `request.messages` V2

`toLLMMessages(context, model)` proyecta user, synthetic, system, shell, assistant, tools y compaction. Cuando se alcanza el step limit añade `MAX_STEPS_PROMPT` como assistant y deshabilita tools.

### Metadata V2

El runner añade explícitamente:

- `x-session-affinity`;
- `X-Session-Id`;
- `x-parent-session-id` cuando existe padre;
- `providerOptions.openai.promptCacheKey` derivado de la Session.

Estas propiedades son confirmadas del **request V2**.

### Continuidad cross-model V2

`to-llm-message.ts` puede reutilizar metadata opaque de reasoning/tool calls/results cuando provider/model coinciden y degradar contenido a formas portables cuando cambian. Los eventos `agent-switched`/`model-switched` son control durable y no se convierten necesariamente en texto visible al modelo.

### Compaction V2

La compaction se proyecta como contexto histórico y sustituye la porción reducida de historia; el baseline de `SessionContextEpoch` evita reenviar system context ya absorbido.

---

## 3. Comparación

| Aspecto | `SessionPrompt` | `SessionRunner` V2 |
|---|---|---|
| System context | `SystemPrompt` + `Instruction` + MCP + skills | `agent.system` + `SystemContext`/epoch |
| Historia | `MessageV2.filterCompactedEffect` + SessionV1-compatible parts | `SessionHistory.entriesForRunner` + SessionMessage V2 |
| Compaction | `packages/opencode/src/session/compaction.ts` | `packages/core/src/session/compaction.ts` |
| LLM execution | `packages/opencode/src/session/llm.ts` | `LLMClient` / `@opencode-ai/llm` |
| Tool state | SessionProcessor + Parts | durable Session events + registry settlement |
| Context source snapshot | no `SessionContextEpoch` obligatorio | sí |
| Estado en `dev` | cableado por producto | implementación V2 coexistente |

## 4. Evolución histórica

Branches como `refactor/session-prompt-parts`, `session-prompt-history`, `system-context`, `context-checkpoint` y las familias Core V2 muestran una descomposición progresiva del monolito. Esa evolución explica el runner V2, pero no autoriza a borrar documentalmente el path legacy-compatible mientras siga compuesto y ejecutable.

## Conclusión

La continuidad de OpenCode debe describirse como una migración con dos mecanismos reales. Ambos reconstruyen el request desde estado persistido, pero difieren en cómo representan contexto privilegiado, historia, compaction y metadata. La documentación debe nombrar siempre el pipeline al que atribuye una propiedad.