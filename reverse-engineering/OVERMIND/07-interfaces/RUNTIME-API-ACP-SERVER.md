# Runtime API, ACP and Server

## Dependency order

No empezar por HTTP. La API estable debe emerger después de Session + RunState + EventPort.

```text
Agent/Core
   ^
Session/Execution runtime
   ^
RuntimeApiPort
   ^
+------+-------+------+
| CLI  | WebUI | ACP  |
+------+-------+------+
```

## RuntimeApiPort mínimo

Operaciones candidatas:

```text
create_session
get_session
list_sessions
execute
cancel_execution
get_execution
subscribe_events
reply_permission
compact_context
reset/new conversation
spawn_child (possibly internal first)
```

Interfaces no reciben Agent object.

## OpenCode server lesson

OpenCode separa authoritative HTTP contract, host integration y generated clients. Para Overmind esto solo merece la complejidad cuando exista WebUI/remote client real.

MVP:

- typed Python RuntimeApiPort;
- in-process CLI adapter;
- tests contractuales.

Luego HTTP adapter.

## ACP

OpenCode demuestra que ACP debe ser un mapper explícito de:

- protocol session <-> runtime Session;
- content <-> canonical/public transcript forms;
- Tool state;
- permission request/reply;
- usage;
- events.

No adaptar ACP directamente a ModelService ni ToolRegistry.

## Event subscription

WebUI/ACP necesitan cursor/reconnect. Stable Event IDs + sequence permiten replay de FACTs; SIGNALs pueden perderse y no requieren historical replay salvo UX específica.

## Security

External server authentication y runtime PermissionService son capas distintas:

- server auth: quién puede llamar a Overmind;
- runtime permission: qué acción puede ejecutar el agent en una Session.
