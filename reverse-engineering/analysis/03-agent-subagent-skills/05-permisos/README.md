# Permisos de agentes y subagents

## Principio general

OpenCode separa:

- la policy declarada por el `Agent.Info`;
- restricciones de la sesión concreta;
- autorización de la llamada que crea/delega al subagent.

Un child no recibe una copia ciega de los permisos del padre.

## Delegation permission

Antes de crear un child, `TaskTool` evalúa el permiso `task` usando como pattern el nombre del subagent.

Esto permite reglas como:

```text
task:explore  -> allow
task:general  -> ask
task:*        -> deny
```

La lista de agentes expuesta al modelo también puede filtrarse por esas reglas, de modo que autorización y discoverability están relacionadas.

**Hecho confirmado.** `TaskTool` aplica la evaluación antes de construir la descripción de agentes disponibles y vuelve a pasar por `ctx.ask()` durante la ejecución salvo bypass explícito.

## Policy del child

`packages/opencode/src/agent/subagent-permissions.ts` contiene la derivación de permisos de sesión del subagent.

**Hecho confirmado.** El objetivo no es reemplazar `agent.permission`, sino añadir una capa de restricciones derivadas del contexto parental.

La propiedad de seguridad importante es:

> una restricción parental relevante no debe desaparecer sólo porque el child use otro Agent.Info.

Al mismo tiempo, no todos los `allow` del padre deben transferirse automáticamente, porque el agente hijo posee su propia capability policy.

## `subagent-permission-inheritance`

Esta branch representa la formalización de la herencia entre parent y child.

La semántica que sobrevive en `dev` es una **herencia restrictiva**, especialmente de denegaciones, en vez de una clonación completa del ruleset.

### Razón arquitectónica

Supóngase:

```text
Parent Agent:
  bash = deny
  webfetch = allow

Child Agent:
  bash = allow
  webfetch = deny
```

Una copia exclusiva del child reabriría `bash`; una copia exclusiva del parent destruiría la especialización del child.

La composición correcta necesita preservar límites del parent mientras mantiene restricciones propias del child.

```text
Effective child permissions
    = child agent policy
    + inherited parent/session restrictions
    + task/depth/runtime restrictions
```

## Restricciones propias de subagent

Históricamente `TaskTool` añade restricciones como:

- deshabilitar `todowrite`;
- deshabilitar `todoread`;
- impedir `task` cuando el child no tiene capacidad de volver a delegar;
- restringir tools marcadas como primary-only.

El detalle exacto ha evolucionado, pero la intención es estable: una child session no debe adquirir accidentalmente herramientas diseñadas sólo para la sesión coordinadora.

## `subagent_depth` como permiso estructural

El límite de profundidad no es sólo una optimización. Es un control de capacidad de delegación.

Con default `subagent_depth = 1`:

```text
root -> child       permitido
child -> grandchild bloqueado
```

Aunque el `Agent.Info` del child permitiera `task`, la restricción estructural puede impedir spawning adicional.

**Hecho confirmado.** El schema documenta explícitamente ese comportamiento.

## `agent-permission-order`

Esta familia evidencia que el **orden de composición** de rulesets es semánticamente relevante.

En un sistema de rules matching, mezclar defaults, user config, agent config y session restrictions en orden incorrecto puede transformar un `deny` en `allow` o viceversa.

**Hecho confirmado a nivel de baseline:** `Agent.state()` y la infraestructura Permission realizan merges explícitos de defaults y configuración; los subagents añaden otra fase de derivación.

**Inferencia prudente sobre la branch:** su propósito es estabilizar precedencia, no introducir una categoría nueva de permisos.

## Bypass por invocación explícita

`TaskTool` contempla `ctx.extra?.bypassAgentCheck` para casos en los que el usuario ya ha seleccionado explícitamente el subagent mediante una surface como `@agent` o un mecanismo equivalente.

**Hecho confirmado.** Ese bypass evita la comprobación redundante de selección/delegación.

**No implica:**

- desactivar `bash`, `read`, `edit`, etc.;
- ignorar `subagent_depth`;
- ignorar permisos internos del Agent.Info.

La misma distinción aparece en la evolución de skills: una activación explícita del usuario puede evitar una aprobación redundante, pero no debe convertirse en un bypass global de policy.

## `skill-mention-bypass` como evidencia paralela

Commit de esta línea restringe el bypass de skills activadas explícitamente al contexto de tool (`context.userRequests`) en vez de contaminar la capa global de permisos.

**Inferencia de diseño común:** OpenCode intenta representar “el usuario pidió explícitamente esta capacidad” como contexto de una operación concreta, no como mutación permanente del ruleset.

## Primary tools

`config.experimental.primary_tools` permite declarar tools que deben reservarse a agentes primary.

En la ejecución de child, estas herramientas se deshabilitan aunque puedan existir globalmente en el registry.

Esto refuerza la separación:

```text
Tool registry       = capacidades instaladas
Agent permission    = capacidades autorizadas por perfil
Session restriction = capacidades autorizadas en este contexto
```

## Threat model reconstruido

Los permisos de subagent parecen defender principalmente contra:

1. **privilege escalation por cambio de agent:** elegir un child más permisivo no debe saltarse límites del padre;
2. **delegation explosion:** recursion ilimitada de subagents;
3. **primary-only leakage:** herramientas de coordinación no deben aparecer en children;
4. **implicit user approval:** una decisión del modelo de delegar no equivale siempre a consentimiento del usuario;
5. **sticky bypass:** una aprobación explícita puntual no debe abrir permisos posteriores no relacionados.

## Hechos e inferencias

### Confirmado

- `Agent.Info` contiene ruleset de permisos.
- `TaskTool` evalúa `task:<agent>`.
- existe derivación específica de permisos de child.
- `subagent_depth` limita spawning.
- existen restricciones de tools primary/subagent.

### Inferido

- La arquitectura está convergiendo hacia tres capas claramente independientes: policy del perfil, restricciones del agregado Session y contexto de autorización de la operación.
- Las branches de orden/herencia son correcciones de composición de esas capas, no intentos de crear un sandbox separado para agents.
