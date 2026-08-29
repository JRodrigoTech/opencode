# 04 — Daemon, service lifecycle, discovery y versionado

## 1. Resumen evolutivo

La línea de daemon/service muestra una progresión clara:

```text
server-discovery
  URL + PID + health
       │
       ▼
daemon-election / client-service
  ownership + lock + registration + auth + version
       │
       ▼
service-* hardening
  restart, directional version policy, authenticated stop, watchdog
       │
       ▼
dev: CLI Daemon
  registro autenticado compacto + health + PID-safety
```

No todos los mecanismos de las ramas intermedias sobreviven literalmente, pero esas ramas revelan los failure modes que el diseño intenta controlar.

---

## 2. Primera generación: `server-discovery`

### Hecho confirmado

`server-discovery` introduce un registro local para localizar un servidor ya arrancado. El contrato inicial gira alrededor de:

- URL;
- PID;
- health probe.

### Problema resuelto

Un cliente local deja de asumir un puerto fijo o de arrancar siempre su propio backend. Puede descubrir una instancia reutilizable.

### Limitaciones de esta generación

URL+PID no bastan para distinguir con seguridad:

- un PID reutilizado por otro proceso;
- un proceso de otra versión;
- un endpoint local no perteneciente a la instancia registrada;
- múltiples contenders arrancando simultáneamente.

Las ramas posteriores atacan exactamente esos casos.

---

## 3. Election y ownership: `daemon-election`

Commit principal inspeccionado:

`c61e93a1f3f6aa5d0f09904c685cd51db0762f4b` — `fix(cli): harden election edge cases`.

### Hechos confirmados

- Un proceso que pierde ownership (`ServiceAlreadyOwned`) sale en vez de competir indefinidamente.
- El retry sólo se aplica a elecciones perdidas, con espera acotada.
- Al decidir si un lock owner está muerto, **sólo `ESRCH` prueba que el PID no existe**.
- `EPERM` u otros errores no autorizan a romper el lock; se interpretan conservadoramente como “puede existir un proceso vivo”.

### Significado

Esto es una elección de seguridad distribuida local: ante incertidumbre, se prefiere no crear dos owners.

---

## 4. `client-service`: discovery como contrato completo

`packages/client/src/effect/service.ts` en branch `client-service` contiene la generación más explícita del protocolo de servicio local.

### Registro

El comentario del propio código define el registration file como contrato completo para consumers:

- `url`;
- `pid`;
- `version`;
- password privado;
- permisos de archivo restringidos.

El cliente no necesita leer la configuración interna del daemon.

### `discover()`

- sólo lectura;
- registro + health check + version gate;
- nunca arranca procesos.

### `incumbent()`

Reconoce un servicio autenticado en un URL esperado incluso durante startup/failure, útil para ownership y coordinación.

### `ensure()`

Semántica idempotente:

1. reutilizar servicio healthy y compatible;
2. detectar failed/waiting;
3. reemplazar versión incompatible;
4. si falta, arrancar contenders detached (`opencode serve --service`);
5. volver a descubrir hasta obtener un owner válido.

La implementación evita matar un contender únicamente porque tarda en arrancar.

### Probe

`/api/health` se consulta usando Basic Auth derivado del registro.

El response se valida contra:

- PID registrado;
- versión registrada;
- estado HTTP (`ready`, `waiting`, `failed`).

### Compatibilidad legacy

Existe reconocimiento explícito de health antiguo `{ healthy: true }`, pero se marca como legacy y queda fuera de algunas operaciones modernas.

---

## 5. Autenticación del stop

### Generación inicial de `client-service`

`POST /api/service/stop` acepta identidad de instancia. Si el endpoint no soporta el stop, algunas versiones podían caer a señales del sistema operativo, revalidando primero el registration para reducir riesgo de PID reuse.

### `authenticated-service-stop`

Commit:

`bae7a954a871591211671465f1fc59dac090c72a` — `fix(client): require authenticated service stop`.

La branch endurece radicalmente la política:

- elimina eviction basada sólo en timeouts;
- elimina fallback automático `SIGTERM`/`SIGKILL` para un servicio que no puede autenticarse;
- si hay registro pero el proceso no responde, `stop()` falla y pide intervención manual;
- 404/405 se consideran “no soporta authenticated stop” y son error para lifecycle moderno;
- response ausente se considera rechazo/incertidumbre, no permiso para señalizar;
- `accepted: true` exige además que el proceso termine dentro de la ventana esperada.

### Interpretación

El principio es: **un PID no es una capability suficiente para matar un proceso**. La prueba de identidad debe provenir del endpoint autenticado.

---

## 6. Restart explícito vs reconnect automático

Branch `service-restart`, commit:

`eae5eaff5db0e66b92452ce3a3c2456364878e61`.

### Hechos confirmados

`Service.restart()` se separa de `Service.start()`/ensure.

- restart es recuperación deliberada;
- reconnect automático debe preferir start/ensure;
- si el servicio acepta shutdown, el replacement se inicia después de observar la terminación;
- el request de stop puede incluir `targetVersion`;
- el fetch de health/stop combina cancelación de Effect con timeout de red;
- un registration observado como stale tras la salida del proceso se trata de forma especial para no retrasar innecesariamente el replacement.

### Cliente UI

La misma branch aborta el event stream y cualquier endpoint-resolution pendiente antes de un reload explícito, ejecuta restart y vuelve a levantar el stream después.

### Razón

Restart tiene un riesgo adicional de stale PID/ownership y no debe convertirse en política de autoreconnect indiscriminada.

---

## 7. Version negotiation: `service-version-guard`

Commit inspeccionado:

`910af9a122cb828abc4a12555facf82c9f88e149`.

### Hecho confirmado

La branch introduce una política **direccional** de incompatibilidad.

Un cliente/launcher antiguo que descubre un servicio más nuevo no debe matar automáticamente ese servicio sólo para imponer su propia versión. La recuperación puede recaer en reiniciar el launcher/cliente con una versión apropiada; la branch usa un exit code dedicado (`75`) como señal de restart acotado del launcher.

### Consecuencia

“Version mismatch” no es una relación simétrica. Quién puede reemplazar a quién depende de cuál lado está desactualizado.

### Inferencia

Esta política intenta impedir guerras de versiones entre varias aplicaciones/clientes locales que comparten el mismo daemon.

---

## 8. Shutdown bounded: `service-shutdown-watchdog`

Commit:

`78ed6b212e01b25c7791be54e683d87419c5f2c2` — `fix(cli): bound managed service shutdown`.

### Hecho confirmado

Al entrar en shutdown de service mode se instala un timer nativo de 10 segundos que fuerza `process.exit(1)` si el teardown no termina.

El comentario es explícito: un timer nativo puede ejecutarse incluso si un finalizer de Effect no completa.

En shutdown normal los watchdogs se limpian.

### Significado

La arquitectura reconoce que structured concurrency/finalizers no deben ser el único mecanismo para garantizar que un daemon deje de existir. El supervisor necesita un límite externo al runtime cooperativo.

---

## 9. Estado vigente: `packages/cli/src/services/daemon.ts`

### Hechos confirmados

`dev` conserva los principios fundamentales en una implementación más compacta:

- registro local del daemon;
- password persistente separado con permisos `0600`;
- endpoint autenticado;
- health probe antes de aceptar una instancia;
- comprobaciones de identidad antes de actuar sobre PID;
- revalidación antes de escalar señales, mitigando PID reuse;
- startup/stop encapsulados en el CLI en vez de obligar a cada cliente a conocer process internals.

### Diferencia respecto a `client-service`

La gran API `discover/incumbent/ensure/restart/stop` de la branch no debe asumirse idéntica en `dev`. Lo que sí sobrevive es el modelo conceptual:

```text
registration -> authenticated health -> validated identity -> lifecycle action
```

---

## 10. Failure modes reconstruidos

Las ramas históricas permiten identificar el threat/failure model real:

| Failure mode | Mecanismo |
| --- | --- |
| dos procesos arrancan a la vez | election/lock + contender policy |
| registration stale | health/probe + identidad |
| PID reutilizado | revalidación antes de señales |
| endpoint pertenece a otro proceso | auth + PID/version response |
| servicio arrancando lento | waiting state, no kill prematuro |
| versión incompatible | version gate / replacement policy |
| cliente viejo frente a server nuevo | version guard direccional |
| stop endpoint no responde | rechazo conservador / manual recovery según generación |
| stop aceptado pero proceso no sale | bounded poll / error |
| finalizer se cuelga | native shutdown watchdog |
| UI reconecta durante restart | abort stream/endpoint resolution primero |

---

## 11. Startup y shutdown como protocolo, no como `spawn()`

### Startup

Un startup correcto requiere:

1. elegir/acquire ownership;
2. crear credencial;
3. levantar listener;
4. publicar registration;
5. responder health con identidad coherente;
6. sólo entonces ser discoverable como `ready`.

### Shutdown

Un shutdown robusto requiere:

1. autenticar la instancia objetivo;
2. solicitar stop o iniciar teardown;
3. dejar de anunciar readiness;
4. liberar recursos/listener;
5. eliminar/invalidar registration;
6. verificar salida del proceso;
7. tener escalación/watchdog sólo bajo reglas que no maten un PID ajeno.

### Inferencia

Las ramas demuestran que el “daemon” debe entenderse como un pequeño protocolo distribuido sobre filesystem + HTTP + process primitives, aunque todos los participantes vivan en una sola máquina.

---

## 12. Hechos vs inferencias

### Confirmado

Election safety, authenticated registration, version checks, restart semantics y watchdog aparecen en código/commits concretos de las branches citadas.

### Inferencia

La simplificación posterior en `dev` parece conservar los invariantes considerados más valiosos y descartar parte de la API de orchestration como complejidad accidental. No hay un documento único que declare esa intención; se deduce comparando las implementaciones.