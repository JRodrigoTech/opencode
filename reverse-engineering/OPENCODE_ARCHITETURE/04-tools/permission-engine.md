# Permission Engine

**Status:** VERIFIED-CODE

## Rule form

Conceptualmente una regla tiene:

```text
permission: string/wildcard
pattern: string/wildcard
action: allow | ask | deny
```

La evaluación usa wildcard en ambos ejes y toma la **última** regla coincidente.

## Multi-pattern ask

Una request puede contener varios patterns. Cada uno se evalúa:

- cualquier `deny` → request denegada;
- todos `allow` → ejecución inmediata;
- al menos un `ask`, sin deny → se publica request y se espera decisión.

## Observable events

- `Permission.Asked`
- `Permission.Replied`

Esto desacopla el policy engine de la UI. CLI/app/desktop/ACP pueden representar la misma decisión mediante distintas interfaces.

## Reject semantics

Reject no solo falla la request actual. Recorre requests pendientes de la misma session y las rechaza también. Es una forma de cortar una cadena de ejecución bloqueada por una decisión del usuario.

## Always semantics

Para cada pattern en `Request.always`, añade:

`{ permission, pattern, action: "allow" }`

Después intenta auto-resolver requests pendientes de esa session cuya totalidad de patterns ya quedan allow.

## `visibleTools`

Esta función solo oculta tools cuando encuentra una regla global (`pattern === "*"`) deny para la permission lógica. Las reglas ask o denies de pattern concreto no tienen por qué ocultar la tool; se evaluarán al ejecutar.

## Security consequence

La superficie visible y la autorización efectiva son capas distintas. Un análisis de seguridad que solo mire el catálogo de tools perdería policies por path/command/agent.

## Source

- `packages/opencode/src/permission/index.ts` — `2e27ff2424dbb000ea9ed7f73471769716ba40a1`
