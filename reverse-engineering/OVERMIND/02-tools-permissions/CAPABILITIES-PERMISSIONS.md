# Capability Resolution and Permissions — mantener dos niveles

## Overmind actual

ToolRegistry es frozen; Plugin composition usa grants explícitos; Workspace limita su scope. No existe general runtime permission broker.

No hace falta introducirlo únicamente para delegar coding a OpenCode.

## OpenCode integration

La capability se puede registrar como una única Tool con configuración/grant estático:

```text
OpenCodePlugin
- executable/transport
- workspace_root
- allowed profiles
- timeout/budget config
 -> registers agent.code
```

El modelo no recibe autoridad para cambiar arbitrary executable/root/policy.

## Dos permission domains

### Outer Overmind policy

Controla:

- si `agent.code` está visible;
- si la invocation está permitida bajo los grants del Plugin/session;
- qué workspace/data puede cruzar el boundary.

Con el runtime actual, gran parte de esta policy puede quedar fija por composition/configuración.

### Inner OpenCode policy

Controla acciones concretas del coding agent:

- read/edit;
- shell;
- web;
- external directories;
- subagents.

No duplicar esas reglas en Overmind.

## ACP permission forwarding

Cuando el adapter use ACP, `requestPermission` puede surfaced a un approver del producto. OpenCode production rechaza si ese handler no existe, una semántica fail-closed que conviene conservar.

## Cuándo crear PermissionService genérico

Solo si múltiples capabilities Overmind requieren `ALLOW | ASK | DENY` runtime con el mismo lifecycle — por ejemplo browser + MCP + external agent + filesystem mutations.

Entonces extraer una policy neutral. No copiar necesariamente la precedence exacta `findLast` de OpenCode.

## Invariant

```text
Plugin construction grant != external agent inner permission
```

Tener permiso para arrancar el agente OpenCode no significa aprobar automáticamente cada mutación que éste solicite.
