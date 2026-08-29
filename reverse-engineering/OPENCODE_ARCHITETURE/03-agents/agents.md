# Agents

**Status:** VERIFIED-CODE

## Agent data model

`Agent.Info` incluye, entre otros, `name`, `description`, `mode`, `native`, `hidden`, sampling (`topP`, `temperature`), `color`, `permission`, `model`, `variant`, `prompt`, provider/model `options` y límite de `steps`.

El campo `mode` separa agentes primarios de subagentes:

- `primary`
- `subagent`
- `all`

## Built-ins

### `build`

Agente primario por defecto orientado a ejecución. Hereda la policy general y no introduce una restricción de edición equivalente a plan.

### `plan`

Agente primario con prompt específico de planificación y policy que restringe edición normal. El runtime habilita excepciones para artefactos de plan y controla las tools de entrada/salida de plan mode según flags/client.

### `general`

Subagent generalista para búsqueda y trabajo multistep. Su policy niega `todowrite` para evitar que manipule el plan de tareas del parent.

### `explore`

Subagent centrado en exploración de codebase. Tiene acceso explícito a herramientas de lectura/búsqueda/shell/web y deniega el resto mediante ruleset restrictivo.

### Hidden agents

- `compaction`
- `title`
- `summary`

Son agentes internos sin exposición normal al usuario; están diseñados para tareas auxiliares del runtime y tienen tool access fuertemente limitado o nulo.

## Default permission scaffold

La configuración base incluye reglas como:

- `*` allow;
- `doom_loop` ask;
- `external_directory` ask salvo rutas internas autorizadas;
- `question` deny;
- `plan_enter` / `plan_exit` deny por defecto;
- lectura de `.env` ask con excepción de `.env.example`.

Después se combinan reglas específicas del agent y configuración del usuario. La semántica final sigue siendo “last matching rule wins”, por lo que el orden importa.

## Custom agents

La config puede definir/agregar agentes y sobrescribir atributos del built-in. Campos configurables incluyen model, variant, prompt, description, mode, hidden, temperature/topP, color, steps, options y permissions.

## Agent generation

Existe un flujo separado que usa un LLM para generar configuración de agente desde una description. No forma parte del agent loop normal; es una utilidad de authoring/configuración.

## Sources

- `packages/opencode/src/agent/agent.ts` — `536a642fe49fb5211e66c2e2ad689856a03254c0`
- `packages/opencode/src/agent/generate.txt` — `774277b0...`
- `packages/opencode/src/permission/index.ts`
