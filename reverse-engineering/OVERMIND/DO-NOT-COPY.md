# What Overmind Should Not Copy from OpenCode

## 1. No construir otro coding agent dentro de Overmind

No copiar el conjunto shell/read/edit/write/apply_patch/grep/glob/LSP/Code Mode/Git workflows para conseguir una capacidad que OpenCode ya ofrece como agente especializado.

Cuando Overmind necesite coding complejo, delegar una misión a OpenCode.

## 2. No replicar los agentes de coding

`build`, `plan`, `explore`, `general` y sus prompts/policies pertenecen al dominio OpenCode. Overmind no necesita mantener copias de esos perfiles para poder utilizarlos.

## 3. No copiar la jerarquía de subagents solo para programar

OpenCode puede lanzar sus propios subagentes dentro de una misión delegada. El parent Overmind no necesita representar cada subagent interno como child Agent propio.

Un futuro Subagent de Overmind debe existir por una necesidad cognitiva general de Overmind, no para reproducir TaskTool.

## 4. No hacer de Session messages el Context canónico

Resume nunca debe convertirse en `load provider messages -> send provider messages`. Canonical ContextUnits y ContextCompiler siguen siendo autoridad.

## 5. No importar Effect/Layer architecture

Es una solución del stack TypeScript de OpenCode. Overmind mantiene ports/composition Python explícitos.

## 6. No copiar el provider registry de OpenCode

El agente OpenCode delegado puede usar sus providers internamente. Overmind mantiene `ModelService -> GenerationExecutor -> ModelBackend` para su propia cognition.

## 7. No copiar el permission engine solo para controlar OpenCode

Usar dos boundaries:

- outer Overmind grant/policy para permitir la delegación y su scope;
- inner OpenCode permissions para las operaciones de coding.

Si múltiples capabilities de Overmind prueban la necesidad de runtime approvals, entonces diseñar una primitive neutral.

## 8. No hacer obligatorio SessionService/EventBus/Persistence para la primera delegación

Una Tool synchronous con `external_session_id` opaque es suficiente para validar el patrón. Extraer infraestructura universal antes de necesitarla contradiría `Simplify first`.

## 9. No copiar Git snapshot/revert como Core

Las mutaciones realizadas por el agente OpenCode son responsabilidad de su runtime. Overmind mantiene la recovery de sus propias Workspace Tools. Un journal transversal solo se justifica con evidencia cross-capability.

## 10. No usar model-ID hacks

OpenCode contiene heurísticas pragmáticas de tool selection. Overmind debe mantener capabilities explícitas donde necesite model feature negotiation.

## 11. No copiar un hook surface interceptivo

Preservar Plugins por public ports. Observers/typed middleware solo ante casos reales.

## 12. No exponer las tools internas de OpenCode una por una al modelo Overmind

Evitar convertir el Tool catalog de Overmind en una réplica del catálogo OpenCode. Preferir una operación de alto nivel:

```text
opencode.delegate(task, ...)
```

Esto reduce tokens de schemas, coupling y decisiones de dominio en el parent.

## 13. No importar transcript/reasoning del agente externo como memoria

Guardar, como máximo, result summary, artifact refs y opaque external session ID. La transcript completa puede permanecer en OpenCode y consultarse on demand.

## 14. No auto-aprobar permisos peligrosos

La production CLI de OpenCode ofrece `--auto`, `--yolo` y `--dangerously-skip-permissions`; el adapter de Overmind no debe convertir estos flags en comportamiento normal. Preferir least privilege y explicit approvals.

## 15. No acoplar Core al wire protocol ACP

ACP pertenece al adapter. El Core ve una Tool/DelegationResult. Si el transporte cambia a CLI JSON, HTTP o otro protocolo, la semántica Overmind no debería cambiar.

## 16. No adelantar infraestructura porque OpenCode sea grande

HTTP server, generated clients, background workers, MCP, provider matrices, persistence avanzada y otros subsistemas deben entrar cuando **Overmind** tenga un caso de uso, no para alcanzar feature parity.
