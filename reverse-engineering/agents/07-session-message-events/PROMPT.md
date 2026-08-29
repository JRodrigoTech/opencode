# Agente 07 — Sessions, messages, events y persistence

Actúa como ingeniero senior de reverse engineering. Analiza todas las branches `session-*`, `message-*`, `protocol-events`, `published-events`, `event-*`, `step-*`, `state-*`, `storage-*`, `sqlite` y equivalentes. Reconstruye la state machine de sesión/turno, modelos de mensaje/part, lifecycle, persistencia, event bus/stream, ordering, replay, recovery, forking, archival, migrations y consistency guarantees.

Compara siempre contra `dev`. Escribe exclusivamente en la branch `reverse-engineering`, bajo `reverse-engineering/analysis/07-session-message-events/`, con subcarpetas por branch o familia demostrablemente equivalente y `README.md` de síntesis. Incluye schemas, paths, símbolos, commits/PRs y separa hechos de inferencias.