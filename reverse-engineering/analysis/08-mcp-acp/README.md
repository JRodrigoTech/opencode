# Reverse engineering de MCP y ACP en OpenCode

## Alcance

Este directorio reconstruye la evolución y la arquitectura observable de las integraciones **Model Context Protocol (MCP)** y **Agent Client Protocol (ACP)** de OpenCode. La baseline funcional usada para el estado vigente es `dev@dc4449df0d52199704ea4989a5a993ebbc605612` (29-08-2026). La documentación se escribió exclusivamente sobre `reverse-engineering`.

La investigación no interpreta el nombre de una branch como prueba suficiente. En este repositorio existen branches antiguas cuyo HEAD ha seguido avanzando o ha absorbido merges de líneas mayores; por ello se distingue entre:

- **Hecho confirmado**: comportamiento observable en `dev`, en un commit concreto o en un ref vivo resoluble.
- **Inferencia**: reconstrucción evolutiva apoyada por varios indicios, pero no demostrable únicamente por una línea de código o commit.
- **Ref histórico degradado**: nombre aparecido en índices de búsqueda de GitHub pero que ya no resuelve como branch viva. No se usa como prueba arquitectónica primaria.

## Documentos

### MCP

- [`mcp/arquitectura-y-lifecycle.md`](mcp/arquitectura-y-lifecycle.md): servicio MCP vigente, transports, conexión, discovery, tools, prompts, resources, namespaces y teardown.
- [`mcp/oauth-recursos-eventos.md`](mcp/oauth-recursos-eventos.md): OAuth, refresh, callback, persistencia de credenciales, resource APIs y eventos de catálogo.

### ACP

- [`acp/arquitectura-sesiones-eventos.md`](acp/arquitectura-sesiones-eventos.md): Agent/Service, lifecycle ACP, sesiones, replay, streaming, permisos y MCP server registration.
- [`acp/mapping-attachments-elicitation.md`](acp/mapping-attachments-elicitation.md): traducción de contenido/attachments, mapping ACP↔OpenCode, elicitation y actividad de subagentes.

### Historia

- [`historico/branches.md`](historico/branches.md): inventario de refs vivos relacionados, SHAs y criterio de agrupación.
- [`historico/generaciones.md`](historico/generaciones.md): generaciones de diseño MCP/ACP y qué ideas llegaron, cambiaron o quedaron experimentales frente a `dev`.

## Conclusiones ejecutivas

### 1. MCP dejó de ser “un conjunto de tools externas” y se convirtió en un subsistema de integración

**Hecho confirmado.** En `dev`, `packages/opencode/src/mcp/index.ts` implementa un servicio Effect con estado por instancia, múltiples transports, lifecycle explícito, OAuth persistente, estados de conexión y acceso uniforme a tools, prompts, resources y resource templates. `packages/opencode/src/mcp/catalog.ts` concentra paginación, tolerancia a servidores defectuosos y adaptación de tool schemas.

**Inferencia.** La evolución observable indica tres etapas: cliente mínimo/eager de la era `v2`, endurecimiento de lifecycle/OAuth y, finalmente, integración del catálogo MCP con API/SDK/TUI y eventos de actualización.

### 2. La conexión remota MCP usa negociación práctica de transport, no un único protocolo rígido

**Hecho confirmado.** La implementación vigente intenta Streamable HTTP y después SSE para servidores remotos. Las conexiones locales usan `StdioClientTransport`. Los errores de autenticación detienen el fallback para producir estados semánticos `needs_auth` o `needs_client_registration`, mientras otros fallos permiten probar el siguiente transport.

### 3. Tool discovery es el único catálogo MCP materializado agresivamente en el runtime

**Hecho confirmado.** Las tools se listan y transforman a definiciones internas y reciben invalidación por `tools/list_changed`. Prompts y resources se consultan de forma lazy sobre clientes conectados. La implementación histórica `v2` hacía discovery eager de tools, prompts y resources tras conectar; `dev` reduce ese acoplamiento.

### 4. OAuth MCP es parte del lifecycle, no un comando lateral

**Hecho confirmado.** `McpOAuthProvider` persiste tokens, información de cliente dinámico, PKCE verifier, state y URL del servidor; rechaza reutilizar credenciales si cambia la URL. El refresh previo a conexión está acotado y existe una evolución específica para cancelar refresh bloqueados mediante `AbortSignal` (`ecf550c8`, `fix(mcp): cancel timed out oauth refresh`).

### 5. Los namespaces MCP están diseñados para evitar colisiones entre servidores

**Hecho confirmado.** Las tools se proyectan como `<server_sanitizado>_<tool_sanitizada>`. Prompts y resources usan claves con namespace de servidor; los resources preservan la URI, escapando delimitadores necesarios. Esta decisión convierte el conjunto de servidores MCP en un catálogo global utilizable por el agente sin perder la identidad de origen.

### 6. ACP es un adaptador de modelo, no el runtime del agente

**Hecho confirmado.** `packages/opencode/src/acp/agent.ts` implementa la interfaz Agent del SDK ACP y delega en `ACPService`. `ACPService` crea/carga/reanuda/forkea/cierra sesiones usando el SDK interno de OpenCode. El agente real, el modelo, las tools y la persistencia siguen perteneciendo a OpenCode; ACP traduce su representación y lifecycle.

### 7. ACP mantiene una correspondencia de sesión externa con sesión OpenCode y reconstruye configuración

**Hecho confirmado.** `newSession`, `loadSession`, `resumeSession` y `unstable_forkSession` usan sesiones persistentes de OpenCode, restauran model/variant/mode desde historial cuando aplica, registran MCP servers recibidos por ACP y mantienen snapshots de catálogo/directorio para construir `configOptions`.

### 8. El streaming ACP se deriva del event stream global de OpenCode

**Hecho confirmado.** `ACPEvent` mantiene una suscripción al stream global, reconecta tras desconexión y traduce `message.part.delta`, `message.part.updated`, `permission.asked` y `session.status`. `runUntilIdle` no completa una petición de prompt hasta observar el estado `idle`, preservando el orden entre chunks y respuesta final ACP.

### 9. Attachments y resources no comparten exactamente el mismo modelo, por lo que existe una capa de proyección

**Hecho confirmado.** `packages/opencode/src/acp/content.ts` traduce texto, imágenes, `resource_link` y embedded resources ACP a parts internos de OpenCode y realiza la conversión inversa para replay. `file://` se proyecta como resource link; imágenes data-URI como `image`; otros data URIs como embedded resources text/blob.

### 10. Elicitation ACP existió como evolución explícita y no forma parte del `dev` observado

**Hecho confirmado.** El commit `a36c8392` (`feat(cli): support acp elicitation`) implementó soporte para `unstable_createElicitation`: convierte formularios internos de OpenCode en schemas ACP y remite el resultado de vuelta a `form.reply`. Una búsqueda de `unstable_createElicitation` en el estado vigente no devuelve implementación equivalente.

**Inferencia.** Elicitation debe considerarse una línea experimental/no consolidada en la baseline, no una capacidad vigente del adaptador ACP.

### 11. La actividad de subagentes requiere una proyección especial en ACP

**Hecho confirmado.** `acp-subagent-events@ca357ee2` añade tracking de child sessions, resuelve cada child hacia la sesión ACP raíz y proyecta los updates con `_meta["opencode/child-session"]`, además de prefijar IDs de tool calls para evitar colisiones.

**Inferencia.** Esta branch revela una tensión estructural: OpenCode modela subagentes como sesiones hijas reales, mientras ACP expone una sesión lógica al cliente. La solución propuesta preserva la sesión ACP raíz y transporta la jerarquía como metadata.

## Modelo mental resumido

```text
Cliente/IDE ACP
    │
    │ ACP requests + sessionUpdate
    ▼
ACP Agent → ACPService → SDK OpenCode → Session/runtime persistente
               │                  │
               │                  └─ event stream global
               │                          │
               └─ registra MCP servers    ▼
                                      ACPEvent → traducción de chunks/tools/permisos

Runtime OpenCode
    │
    ├─ tools internas
    └─ MCP Service
          ├─ local: stdio
          ├─ remote: Streamable HTTP → fallback SSE
          ├─ OAuth / refresh / callback
          ├─ McpCatalog: tools/prompts/resources/templates
          └─ events de invalidación/cambios
```

## Referencias de código principales

- `packages/opencode/src/mcp/index.ts`
- `packages/opencode/src/mcp/catalog.ts`
- `packages/opencode/src/mcp/oauth-provider.ts`
- `packages/opencode/src/mcp/auth.ts`
- `packages/opencode/src/server/routes/mcp.ts`
- `packages/opencode/src/acp/agent.ts`
- `packages/opencode/src/acp/service.ts`
- `packages/opencode/src/acp/event.ts`
- `packages/opencode/src/acp/content.ts`
- `packages/opencode/src/acp/permission.ts`

## Limitaciones arqueológicas

El repositorio contiene muchas branches de trabajo de vida larga. Un `compare dev...branch` puede incluir cientos o miles de commits no relacionados porque las líneas divergieron durante meses. Por tanto, los comparativos históricos se normalizan así:

1. `dev` define el comportamiento actual.
2. Se usa el HEAD vivo para saber que la branch sigue existiendo.
3. Se usan commits semánticos concretos para atribuir una decisión a MCP/ACP.
4. Cuando una branch histórica se basa en `v2`, se compara contra esa línea o su merge-base para evitar atribuir reescrituras generales al protocolo.
5. Si el HEAD ya no representa el nombre original de la branch, se documenta la discrepancia en vez de forzar una conclusión.