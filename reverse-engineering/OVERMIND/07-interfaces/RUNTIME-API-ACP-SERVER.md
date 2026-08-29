# ACP tiene dos direcciones distintas

## 1. Overmind como ACP client de OpenCode

**Puede ser útil temprano.**

OpenCode production ofrece `opencode acp`, un server ACP por stdio. Un `OpenCodeAgentAdapter` dentro de un Plugin Overmind puede actuar como client y consumir:

- session lifecycle;
- prompt;
- resume/fork;
- cancel;
- permission request/reply;
- agent events;
- model/mode config.

Este ACP es **transport interno de una capability**. Agent/Core de Overmind no tiene por qué conocerlo.

```text
Overmind Tool
 -> OpenCodeAgentAdapter
 -> ACP client
 -> `opencode acp`
```

## 2. Overmind como ACP server

**Sigue deferred.**

Exponer Overmind a editores/clientes ACP requiere una Runtime API estable de Overmind y mapping de sus propias session/content/tool/permission/event semantics.

No confundir ese trabajo con usar OpenCode como backend.

## RuntimeApiPort

Implementarlo solo cuando exista una second surface real (WebUI, ACP server, remote client) o operations cross-caller que lo exijan.

Entonces:

```text
Overmind Core/Runtime
   ^
RuntimeApiPort
   ^
CLI / WebUI / ACP-server adapters
```

La OpenCode capability puede seguir siendo un Plugin detrás de ese runtime.

## Security

Transport authentication y capability permission siguen separados. Del mismo modo, la seguridad del proceso OpenCode no se obtiene simplemente por usar ACP sobre stdio.
