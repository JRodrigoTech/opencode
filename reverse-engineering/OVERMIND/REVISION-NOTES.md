# Revision Notes — corrección de orientación

## Motivo

La primera versión de `reverse-engineering/OVERMIND/` separaba explícitamente ambos proyectos, pero seguía promoviendo demasiados mecanismos de OpenCode a un target runtime de Overmind: SessionService, EventPort, PermissionService, child sessions, MutationJournal y otras primitives aparecían como una secuencia de implementación casi obligatoria.

Eso era una lectura demasiado influida por la madurez de OpenCode.

## Corrección

La revisión actual aplica con más rigor las reglas del propio Overmind:

- simplify first;
- Core pequeño;
- capabilities independientes;
- public ports solo cuando exista necesidad real;
- composición antes que inheritance/frameworks;
- promover una primitive neutral solo con evidencia.

## Cambios de decisión

| Tema | Antes | Ahora |
|---|---|---|
| Coding agent | parecía requerir ampliar Overmind tools/runtime | **delegar a OpenCode** |
| Subagents | child sessions P1 de Overmind | **OpenCode internal subagents para coding; subagents Overmind solo si hay caso propio** |
| SessionService | target temprano | **condicional a persistence/multi-interface/subagents propios** |
| EventPort | P0/P1 | **implementar cuando una capability Overmind lo necesite** |
| PermissionService | P1 general | **outer grant + OpenCode inner policy; genericizar después si se repite** |
| MutationJournal | P1/P2 | **no necesario para mutaciones delegadas a OpenCode; defer generalización** |
| ACP | interfaz Overmind tardía | **dos direcciones: ACP client a OpenCode puede ser temprano; ACP server Overmind sigue deferred** |
| Provider stack | referencia | **claramente propiedad separada de cada runtime** |

## Nueva tesis

```text
Overmind = cognition/orchestration + universal contracts mínimos
OpenCode = specialized software-engineering agent capability
```

No se pretende reemplazar las Tools/Plugins de Overmind. Una delegación a OpenCode es una capability adicional para tareas que requieren un coding loop especializado.

## Evidencia nueva incorporada

La revisión verificó directamente en OpenCode production:

- `opencode run` soporta non-interactive single prompt, raw JSON events, `--session`, `--fork`, `--agent`, `--model`, `--dir`;
- `opencode acp` inicia un ACP server sobre stdio;
- el ACP Agent implementa new/load/list/resume/close/fork session, prompt, cancel y mode/model/config operations;
- ACP permission handling publica allow-once/allow-always/reject y falla cerrado si no existe `requestPermission`;
- `TaskTool` demuestra que OpenCode puede poseer sus propias child sessions/subagents detrás de la misión delegada.

## Consecuencia documental

Los documentos de runtime/state/tools se mantienen porque contienen referencias útiles, pero ahora están marcados como **conditional extraction**, no como blueprint obligatorio.
