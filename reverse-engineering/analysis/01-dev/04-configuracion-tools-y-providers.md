# 04 — Configuración, tools y providers

## Configuración efectiva

Path principal: `packages/opencode/src/config/config.ts`.

**[CONFIRMADO]** La configuración de OpenCode se compone a partir de varias fuentes, con merge y precedencia. Entre las fuentes observadas están:

- configuración remota / well-known;
- configuración global del usuario;
- archivo indicado explícitamente;
- configuración del proyecto;
- directorios `.opencode`;
- contenido suministrado por variables de entorno;
- configuración de organización/console;
- política/configuración administrada.

**[CONFIRMADO]** El servicio está ligado al contexto de instance. El mismo proceso puede, por tanto, resolver configuraciones distintas para directorios/proyectos distintos.

### `.opencode` como extension root

**[CONFIRMADO]** Los directorios de configuración no contienen sólo valores escalares. Se exploran para descubrir artefactos como commands, agents y plugins; el bootstrap también contempla preparar dependencias necesarias para esas extensiones.

**[INFERENCIA]** Configuración y extensibilidad forman un único boundary operativo: cambiar de instance puede modificar no sólo parámetros, sino el conjunto de comportamiento cargado.

## Agents

Paths relevantes:

- `packages/opencode/src/agent/agent.ts`
- `packages/opencode/src/config/agent.ts`
- `packages/opencode/src/agent/subagent-permissions.ts`

**[CONFIRMADO]** Un agente no es sólo un nombre de prompt. Participa en la selección de modelo, instrucciones, permisos y disponibilidad/comportamiento de herramientas. Existen agentes internos auxiliares para tareas como compaction, summary o exploración además de agentes configurables por el usuario/proyecto.

**[INFERENCIA]** Agent actúa como una policy bundle aplicada sobre el runtime de sesión: configura cómo razonar y qué capacidades quedan disponibles sin introducir un runtime separado.

## Tool registry

Path principal: `packages/opencode/src/tool/registry.ts`.

El árbol `packages/opencode/src/tool/` muestra built-ins para, entre otros:

- lectura/escritura/edición;
- glob/grep;
- shell;
- apply patch;
- LSP;
- web fetch/search;
- task/subagent;
- skills;
- todos/questions;
- plan mode;
- code mode;
- truncation.

**[CONFIRMADO]** `ToolRegistry` es el catálogo base, mientras `packages/opencode/src/session/tools.ts` resuelve el conjunto efectivo para una ejecución concreta.

## Resolución de tools por sesión

**[CONFIRMADO]** El conjunto enviado al modelo depende de más factores que la existencia de un tool registrado. Durante la resolución intervienen:

- agente activo;
- configuración;
- capacidades/compatibilidad del modelo;
- permisos;
- tools de plugins;
- tools descubiertas vía MCP;
- condiciones/flags específicos del runtime.

**[INFERENCIA]** La arquitectura distingue correctamente entre **tool definition** y **tool exposure**. El registry define capacidades potenciales; `SessionTools` produce la capability surface efectiva de cada turn.

## Lifecycle de una tool call

La combinación de `session/tools.ts`, `session/processor.ts` y los propios tools permite reconstruir este flujo:

```text
modelo emite tool call
  -> SessionProcessor crea/actualiza part
  -> parámetros se validan contra schema
  -> wrapper de tool construye execution context
  -> permission evaluation / posible approval
  -> execute()
  -> output + metadata / error
  -> truncation si corresponde
  -> actualización de part
  -> resultado vuelve al siguiente paso LLM
```

**[CONFIRMADO]** Los estados de tool call son persistibles/observables y no quedan ocultos dentro de una única llamada al provider.

## Permisos

Paths:

- `packages/opencode/src/permission/index.ts`
- `packages/opencode/src/permission/arity.ts`
- `packages/opencode/src/agent/subagent-permissions.ts`
- wrappers en `session/tools.ts` y tools sensibles como shell/edit/write.

**[CONFIRMADO]** La autorización es parte del tool runtime. Las reglas pueden producir allow/deny/ask y pueden variar por tool, patrón/argumentos, agente y contexto heredado.

**[CONFIRMADO]** La TUI expone una superficie específica para responder solicitudes de permiso, lo que demuestra que approval es un protocolo entre backend y cliente, no un diálogo implementado dentro del tool.

**[INFERENCIA]** El permission system constituye el security boundary más importante entre razonamiento no confiable del modelo y efectos locales sobre filesystem/procesos.

## Providers y modelos

Path principal: `packages/opencode/src/provider/provider.ts`.

**[CONFIRMADO]** `Provider` mantiene una abstracción de provider/model que combina:

- catálogo/model metadata;
- configuración del usuario/proyecto;
- autenticación;
- plugins e integraciones provider-specific;
- creación/resolución del modelo ejecutable;
- aliases/opciones/variantes.

El gran tamaño del módulo y la presencia paralela de `provider/transform.ts`, `provider/auth.ts` y plugins específicos muestran que la compatibilidad de providers sigue siendo una responsabilidad considerable del composition layer.

## Capa LLM de sesión

`packages/opencode/src/session/llm.ts` se sitúa entre SessionPrompt y el provider concreto.

**[CONFIRMADO]** Esta capa recibe mensajes normalizados, modelo, agente, system prompt y tools; construye opciones de generación/streaming y aplica transformaciones/hook de plugins/provider antes de devolver el stream al processor.

El subdirectorio `packages/opencode/src/session/llm/` contiene caminos `ai-sdk.ts`, `native-request.ts`, `native-runtime.ts` y `request.ts`.

**[INFERENCIA]** `dev` está desacoplando gradualmente la ejecución LLM del AI SDK genérico para permitir paths nativos/protocol-aware cuando aportan mejor control sobre streaming, reasoning o compatibilidad.

## `@opencode-ai/llm`

`packages/llm/package.json` expone explícitamente providers y protocolos.

**[CONFIRMADO]** La separación provider/protocol es importante: un provider describe endpoint/autenticación/selección, mientras un protocol adapter modela el wire format y streaming de familias como OpenAI Responses, Anthropic Messages o Gemini.

**[INFERENCIA]** Este package representa un boundary más estable que el antiguo `Provider` monolítico y permite reutilizar protocolos entre providers compatibles.

## System prompt y provider-specific behavior

Paths relevantes:

- `packages/opencode/src/session/system.ts`
- `packages/opencode/src/session/instruction.ts`
- `packages/opencode/src/session/prompt/*.txt`

Existen prompts específicos para familias/modelos como Anthropic, GPT/Codex, Gemini, Kimi y otros, además de plan/compaction/reminder prompts.

**[CONFIRMADO]** La construcción del contexto del modelo es dinámica y dependiente de modelo/agente. Las instrucciones de entorno/proyecto se añaden al system context junto con capacidades disponibles.

## Boundary resumido

```text
Config(instance)
    |
    +--> Agent policy
    +--> Provider/model resolution
    +--> Plugin/MCP extensions
               |
               v
        SessionTools effective set
               |
               v
          SessionPrompt
               |
               v
          Session LLM adapter
               |
               v
        @opencode-ai/llm protocol
               |
               v
            Provider API
```

## Conclusión

**[INFERENCIA]** Config, Agent, Tool y Provider no son subsistemas independientes en tiempo de ejecución: forman una cadena de policy resolution por turn. El punto de convergencia es `SessionPrompt`, que transforma contexto de instance + agent policy + model capabilities + permissions en una petición concreta al modelo.