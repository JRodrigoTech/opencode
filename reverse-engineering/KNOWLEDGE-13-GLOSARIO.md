# 13 — Glosario de OpenCode

## Agent
Perfil declarativo que configura cómo ejecutar una Session: prompt, modelo/variant, sampling, steps y permissions. No es la unidad principal de persistencia.

## Child Session
Session con relación `parentID`. Es la representación fundamental usada para subagentes.

## Compaction
Transformación del historial/contexto para conservar información útil dentro del presupuesto de tokens sin reenviar todo el transcript original.

## Context
Conjunto efectivo de system/instructions/environment/history/skills/MCP y otros elementos que se prepara para un model turn.

## Core V2
Arquitectura más nueva en `packages/core` con runtime, Session/event/context services y boundaries más explícitos. Coexiste con paths todavía activos de `packages/opencode`.

## Durable event
Hecho persistido que puede participar en ordering/replay/recovery. No debe confundirse con cada delta live del streaming.

## Effect
Librería/paradigma usado para modelar services, dependencies, effects, errors, resources y scopes.

## EventV2
Runtime de eventos durables del Core. Usa secuencia por aggregate y projectors dentro del modelo persistente auditado.

## GlobalBus / global event stream
Canal de publicación live usado por clientes/adapters. Publicado no implica automáticamente durable.

## LLMEvent
Vocabulario normalizado de streaming entre provider layer y Session runtime: texto, reasoning, tools, lifecycle, usage y errores.

## Location
Contexto asociado a un directorio/proyecto. Permite materializar services scoped como filesystem/config/tools para ese entorno.

## LayerNode
Representación declarativa del grafo Effect: service, implementación, dependencies, tags, groups, unbound ports y replacements.

## MCP
Model Context Protocol. OpenCode actúa como cliente de servidores que aportan tools, prompts y resources.

## ACP
Agent Client Protocol. OpenCode expone/adapta su runtime para clientes/IDEs ACP; ACP no sustituye a Session/runtime.

## Part
Unidad estructurada dentro de mensajes/ejecución: text, reasoning, tool, etc. Permite modelar lifecycle más rico que `role + string`.

## Permission ruleset
Lenguaje ordenado de reglas `permission + pattern -> allow|deny|ask`. En la baseline auditada, la última regla coincidente gana.

## Projector
Componente que transforma durable events en read models/proyecciones persistentes.

## Provider
Configuración/fachada de un proveedor o deployment: endpoint, auth, defaults, route. No es necesariamente sinónimo de protocol.

## Protocol adapter
Traduce `LLMRequest` al wire format de una API y sus chunks/eventos de vuelta a `LLMEvent`.

## Route
Composición que conecta un modelo con protocol, endpoint, auth, transport y defaults.

## Session
Unidad principal de continuidad del agente. Agrupa historia, model turns, tools, estados, compaction, eventos y relaciones child/fork.

## SessionProcessor
Reducer del stream en el path legacy-compatible/product: convierte `LLMEvent` en messages/parts, tool states, usage, errores y decisiones de continuidad.

## SessionPrompt
Orquestador vigente del path principal de `packages/opencode`: reconstruye estado/contexto, resuelve agent/model/tools, llama LLM y continúa el loop.

## SessionRunner
Runner de la arquitectura Core V2. No debe confundirse automáticamente con el `SessionPrompt` default del producto.

## Skill
Paquete de instrucciones/contexto especializado. Se descubre y, normalmente, se activa de forma lazy; no es un agent ni un subagent.

## Step
Boundary lógico de una fase de ejecución que puede incluir model invocation, reasoning, tools, settlement, usage, retry/continuation.

## Tool
Capability tipada que el modelo puede invocar. Tiene schema y ejecución controlada por runtime/permissions.

## ToolPart
Representación persistible/observable de una tool call con estados como pending, running, completed y error.

## V1 / legacy-compatible
Conjunto de surfaces y paths anteriores que siguen siendo reales en `dev` durante la migración. “Legacy-compatible” no significa “código muerto”.

## V2
Dirección arquitectónica hacia boundaries más explícitos, durable state y packages especializados. Algunas partes ya están en `dev`, otras son más completas en branches `v2/beta`.