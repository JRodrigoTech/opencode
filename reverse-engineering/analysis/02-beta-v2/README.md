# Ingeniería inversa: familia beta / v2 / 2.0

Este directorio reconstruye la evolución de las branches `beta`, `v2`, `2.0` y `opencode-2-0` de OpenCode y las compara contra `dev` como baseline vigente.

> Fecha de corte del análisis: 2026-08-29.

## Convenciones de certeza

- **HECHO**: comprobado directamente mediante historia Git, PRs, commits, código o especificaciones presentes en el repositorio.
- **INFERENCIA**: interpretación arquitectónica sustentada por varios hechos, pero no declarada literalmente por los autores.
- **DESCARTADO / REEMPLAZADO**: una forma concreta de implementación ya no está presente en las ramas posteriores analizadas; no implica necesariamente que todo el concepto haya sido abandonado.

## Resultado ejecutivo

La conclusión principal es que las cuatro branches no representan cuatro arquitecturas independientes.

| Branch | Clasificación | Relación comprobada | Lectura arquitectónica |
| --- | --- | --- | --- |
| `beta` | canal de integración/distribución de V2 | idéntica a `v2` en `106629aa118086be7def6123241a9bf056ba77b6` | V2 preparado para desktop/web/release y validación de integración |
| `v2` | arquitectura alternativa/futura de gran alcance | idéntica a `beta`; diverge de `dev` | separación `core` / `server` / `protocol` / `schema` / `client` / `cli` / `tui`, runtime y persistencia rediseñados |
| `2.0` | experimento puntual previo | su HEAD `7a6ce05d...` es ancestro de `beta/v2`; PR #22335 fue merged | exploración del modelo de sesión mediante `SessionEntry`, no un rewrite completo |
| `opencode-2-0` | prototipo de portabilidad/runtime previo | diverge desde `dev`; PR #16918 cerrado sin merge como unidad | desacoplar OpenCode de Bun en server, procesos, instalación y Node runtime; luego descompuesto en PRs menores |

### Familia equivalente confirmada: `beta` + `v2`

**HECHO.** `git compare beta...v2` devuelve estado `identical`, 0 commits ahead y 0 behind. Las dos refs apuntan a:

```text
106629aa118086be7def6123241a9bf056ba77b6
feat(infra): deploy beta web app with SST (#46086)
```

**HECHO.** El PR upstream #41627 se titula `chore: build beta from v2` y declara que construye `beta` desde `v2` y PRs beta de V2. El PR #41626 publica el desktop beta empaquetando `@opencode-ai/cli@next`.

**Conclusión.** `beta` debe leerse como el canal de integración/release de la arquitectura `v2`, no como una arquitectura paralela independiente.

## Línea evolutiva reconstruida

```text
marzo 2026
  |
  +-- opencode-2-0
  |     PR #16918 (cerrado sin merge como unidad)
  |     objetivo: Node + desacoplar Bun + portabilidad servidor/procesos
  |         |
  |         +-- extracción y merge de piezas a dev
  |             #18308 npm/arborist
  |             #18324 Node entrypoint/build
  |             #18327 OAuth sin Bun.serve
  |             #18328 misc/Windows
  |             #18335 Hono Node server + WS
  |
abril 2026
  |
  +-- 2.0
  |     PR #22335 merged
  |     exploración SessionEntry / modelo de sesión
  |     la forma concreta SessionEntry no sobrevive
  |
  +-- evolución de dev y múltiples líneas v2-*
  |
agosto 2026
  |
  +-- v2 == beta
        arquitectura V2 integral y canal beta
        core + schema + protocol + server + client + cli + tui
        persistencia durable/eventos/inbox
        AI stack @opencode-ai/ai
```

## Comparación arquitectónica de alto nivel

### `dev` actual

`dev` ya contiene varias piezas que surgieron durante esta evolución: `packages/core`, `packages/server`, `packages/cli`, `packages/client`, `packages/protocol`, `packages/schema`, abstracciones Node/Bun y un LLM layer schema-first. Sin embargo, el entrypoint principal del workspace continúa siendo:

```json
"dev": "bun run --cwd packages/opencode --conditions=browser src/index.ts"
```

Esto mantiene a `packages/opencode` como centro operativo del producto vigente.

### `beta/v2`

El entrypoint raíz cambia a:

```json
"dev": "bun run --cwd packages/cli --conditions=browser src/index.ts"
```

El paquete `@opencode-ai/cli` compone `client`, `server`, `tui`, `schema`, `plugin` y `util`; el runtime vive en `@opencode-ai/core`; el transporte HTTP se expresa a través de `@opencode-ai/protocol` y `@opencode-ai/server`; y las formas públicas/durable events viven en `@opencode-ai/schema`.

**INFERENCIA fuerte.** La dirección de V2 es convertir el antiguo paquete `opencode` en una composición de subsistemas explícitos y reutilizables, reduciendo el acoplamiento entre dominio, transporte, UI y runtime.

## Matriz por subsistema

| Área | `dev` | `beta/v2` | Evolución observada |
| --- | --- | --- | --- |
| entrypoint | `packages/opencode` | `packages/cli` | V2 hace real la extracción del CLI |
| runtime agente | lógica todavía repartida entre legado y `core` | `core` como autoridad semántica de ejecución | fuerte consolidación en Core |
| sesión | modelo legacy + evolución parcial | inbox durable, ejecución separada de admisión, claims y replay | rewrite semántico |
| persistencia | SQLite/Drizzle y capas Effect en evolución | eventos durables + proyecciones + inbox + recovery explícito | event-oriented persistence más rigurosa |
| tools | contratos históricos y nuevas capas | único `Tool.make`, registro scoped, snapshot por request | normalización de tool runtime |
| providers/AI | `@opencode-ai/llm` + AI SDK/adapters | `@opencode-ai/ai` schema-first, LLM + Image + routing | expansión del low-level AI stack |
| policy providers | configuración/allowlists históricas | `experimental.policies`, `provider.use`, last-match-wins | autorización desacoplada de configuración |
| server | paquete nuevo pero `packages/opencode` aún central | `server` separado, HttpApi/Protocol como boundary | separación de dominio y wire protocol |
| SDK/client | legacy `@opencode-ai/sdk` aún presente | `@opencode-ai/client`, clientes derivados de HttpApi | reemplazo progresivo de SDK legacy |
| TUI | transición entre legacy y paquete extraído | `packages/tui` consumido por `packages/cli` | boundary de UI más limpio |
| desktop | integra runtime vigente | beta empaqueta CLI V2 `@next` y lifecycle V2 | V2 se prueba como producto distribuible |
| runtime OS | Bun dominante con adaptaciones Node | conditional imports Bun/Node/workerd en Core | portabilidad elevada a contrato |

## Qué conceptos llegaron posteriormente a `dev`

### Confirmados

1. **Portabilidad Node y eliminación de dependencias Bun específicas en puntos críticos.** La línea `opencode-2-0` fue descompuesta en PRs merged (#18308, #18324, #18327, #18328, #18335). La intención de portabilidad dejó de ser experimental.
2. **Separación de paquetes `core`, `server` y `cli`.** Todos existen hoy en `dev`.
3. **Conditional imports para runtime.** `packages/core/package.json` de `dev` ya selecciona implementaciones `bun` / `node` para SQLite, PTY y filesystem.
4. **LLM layer schema-first/provider-neutral.** `dev` contiene `@opencode-ai/llm`; V2 lo expande/renombra a `@opencode-ai/ai`.
5. **Protocol/schema boundaries.** `dev` ya dispone de paquetes separados de protocolo/esquema aunque V2 los usa de forma más completa.

### No absorbidos todavía en la misma forma

1. El root de `dev` aún arranca por `packages/opencode`; V2 arranca por `packages/cli`.
2. `dev` conserva `@opencode-ai/llm`; V2 usa `@opencode-ai/ai` e incorpora generación de imágenes como primitive de primer nivel.
3. `dev` todavía mantiene el SDK legacy (`packages/sdk/js`, `@opencode-ai/sdk`) mientras V2 orienta consumidores hacia `@opencode-ai/client` generado desde `HttpApi`.
4. El contrato de sesión V2 (`session.inbox.*`, claims, log durable, delivery `steer/queue`) no aparece como tal en `dev` actual.

## Ideas descartadas o reemplazadas

### `SessionEntry` de `2.0`

**HECHO.** El PR #22335 creó `packages/opencode/src/v2/session-entry.ts` y retiró `packages/opencode/src/v2/message.ts`, introduciendo `SessionEntry.User`, `Synthetic`, `Request`, `Text`, `Reasoning`, `Tool`, `Complete` y estados de tool.

**HECHO.** El archivo `packages/opencode/src/v2/session-entry.ts` ya no existe ni en `dev` ni en `beta/v2`.

**Conclusión.** La implementación concreta de un stream único de `SessionEntry` fue reemplazada. V2 posterior conserva la preocupación por identidad durable, eventos y herramientas, pero mediante un modelo más explícito de Session + durable events + projections + messages/inbox.

### `opencode-2-0` como mega-branch

**HECHO.** PR #16918 no se fusionó como bloque. Su propio historial contiene commits `extract ... into #...`, y PR #18335 declara explícitamente que es la porción de server restante después de extraer SQLite, process, OAuth, Node entrypoint y fixes a PRs independientes.

**Conclusión.** Se descartó la estrategia de integrar la mega-branch como una sola unidad, no sus objetivos técnicos principales.

## Boundaries V2 reconstruidos

```text
                 +----------------------+
                 | CLI / TUI / Desktop  |
                 +----------+-----------+
                            |
                       @opencode-ai/client
                            |
                 +----------v-----------+
                 |       Protocol       |
                 | HttpApi + wire types |
                 +----------+-----------+
                            |
                 +----------v-----------+
                 |        Server        |
                 | HTTP/SSE/connection  |
                 +----------+-----------+
                            |
                 +----------v-----------+
                 |         Core         |
                 | session/runtime/tools|
                 | persistence/plugins  |
                 +----+------------+----+
                      |            |
              +-------v--+     +---v------------+
              | Schema   |     | AI / Providers |
              | durable  |     | @opencode-ai/ai|
              +----------+     +----------------+
```

La especificación V2 declara esta autoridad de forma explícita:

- Protocol: operaciones HTTP y errores de transporte.
- Schema: shapes públicas y durable event payloads.
- Core: runtime y persistencia.
- Server: composición y wire delivery.

## Documentos de este análisis

- [`familia-v2-beta/README.md`](./familia-v2-beta/README.md): arquitectura V2/beta, runtime, persistencia, tools, providers, protocol, UI y diferencias contra `dev`.
- [`2.0/README.md`](./2.0/README.md): experimento `SessionEntry`, alcance real y qué sobrevivió conceptualmente.
- [`opencode-2-0/README.md`](./opencode-2-0/README.md): prototipo Node/Bun, extracción de cambios y relación con `dev`.

## Evidencia primaria destacada

### Branches / commits

- `dev`: `dc4449df0d52199704ea4989a5a993ebbc605612` al corte.
- `beta` = `v2`: `106629aa118086be7def6123241a9bf056ba77b6`.
- `2.0`: `7a6ce05d0939826aa6c8e1c481489a713b2d633f` — `2.0 exploration (#22335)`.
- `opencode-2-0`: `3eeeec359acda9f14334cb420af84f31bee95f11`.

### PRs upstream relevantes

- #16918 `opencode 2-0` — cerrado sin merge.
- #18308 `replace BunProc with Npm module using @npmcli/arborist` — merged.
- #18324 `add Node.js entry point and build script` — merged.
- #18327 `replace Bun.serve with Node http.createServer in OAuth handlers` — merged.
- #18328 `miscellaneous small fixes` — merged.
- #18335 `replace Bun serve with Hono node adapters` — merged.
- #22335 `2.0 exploration` — merged.
- #40721 `V2 migration` — branch/migration integration work.
- #40723 `migrate v1 data to v2`.
- #41626 `publish v2 beta desktop`.
- #41627 `build beta from v2`.

### Paths V2 de autoridad

- `specs/v2/README.md`
- `specs/v2/session.md`
- `specs/v2/tools.md`
- `specs/v2/event-stream-architecture.md`
- `specs/v2/provider-policy.md`
- `specs/v2/catalog-config-plugin-lifecycle.md`
- `packages/core/package.json`
- `packages/server/package.json`
- `packages/cli/package.json`
- `packages/ai/README.md`

## Hipótesis abiertas

1. **INFERENCIA:** V2 parece la culminación de varias líneas previas (`opencode-2-0`, `2.0`, extracción Effect/core/protocol) más que una continuación lineal de una sola branch. La genealogía Git confirma ancestría para `2.0`, pero la equivalencia conceptual con `opencode-2-0` se debe reconstruir a través de PRs extraídos.
2. **INFERENCIA:** `packages/opencode` probablemente actúa en `dev` como compatibility shell mientras los subsistemas nuevos maduran. La existencia simultánea de `core/server/cli` y el entrypoint legacy apoya esta lectura, pero no se encontró una declaración única que lo nombre formalmente así.
3. **INFERENCIA:** el objetivo de V2 no es solamente “v2 API”, sino establecer una arquitectura multi-runtime con dominio durable y protocolos generables. Esto se deduce de los boundaries, conditional imports, specs de replay y sustitución del SDK.
