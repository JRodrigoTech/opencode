# What Overmind Should Not Copy from OpenCode

## 1. No hacer de Session messages el Context canónico

OpenCode está optimizado para conversaciones/tool parts persistentes. Overmind ya posee una abstracción cognitiva más explícita. Mantenerla.

**Reject:** cualquier diseño donde resume sea simplemente cargar `messages[]` y enviarlos al provider.

## 2. No importar Effect/Layer architecture

Effect resuelve necesidades del stack TypeScript de OpenCode. Overmind ya tiene composition root y ports Python explícitos. Copiar la tecnología añadiría complejidad sin trasladar el valor conceptual.

## 3. No construir un provider registry masivo prematuramente

`ModelService({"primary": target})` ya admite named targets. Añadir backends concretos cuando exista necesidad. Discovery/profile/router global debe requerir un caso real.

## 4. No usar model-ID hacks para capabilities

OpenCode contiene gating pragmático como `gpt-* -> apply_patch` frente a edit/write. Overmind debe preferir `ModelCapabilities` explícitas y capability negotiation.

## 5. No crear un plugin system interceptivo sin límites

OpenCode permite hooks que transforman tool definitions/messages/text/env. Overmind ha decidido que plugins extienden mediante public ports y no mutan internals. Mantener esa propiedad.

Si hace falta middleware, que sea:

- contract-specific;
- typed;
- ordered;
- bounded;
- auditable;
- sin acceso a secrets/context ajeno salvo grant.

## 6. No heredar automáticamente poderes a subagents

Child execution debe recibir grants calculados explícitamente. Parent capability no implica child capability.

## 7. No acoplar reversibility a Git

OpenCode usa snapshots/patches con su workspace model. Overmind debe abstraer mutation records y permitir backend Workspace/FS específico, especialmente por su foco Windows.

## 8. No meter client-specific conditions en Core

El capability set no debería preguntar “¿estoy en CLI/WebUI?” mediante globals. La surface debe suministrar capabilities/approver ports explícitos al runtime.

## 9. No mezclar retry técnico con agent continuation

Overmind ya hace esto mejor: GenerationExecutor distingue retry físico, length continuation y recovery. No regresionar a un loop genérico que los confunda.

## 10. No romper ProtocolUnit atomicity

OpenCode aporta ideas de lifecycle, pero Overmind debe conservar su regla fail-closed: structured ToolCall truncada no ejecuta nada y una Tool exchange completa sigue siendo unidad de protocolo.

## 11. No adelantar infraestructura

No implementar Kafka, distributed workers, hot plugin reload, service discovery ni generic DI porque OpenCode tenga múltiples interfaces. El diseño de Overmind pide la primitive mínima cuando una capability real lo demuestra.
