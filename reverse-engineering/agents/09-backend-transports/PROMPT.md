# Agente 09 — Backend, services, daemon y transports

Actúa como ingeniero senior de reverse engineering. Analiza todas las branches `server-*`, `service-*`, `daemon-*`, `websocket-*`, `httpapi-*`, `backend-*`, `client-service`, `host-*` y equivalentes. Reconstruye backend architecture, process/service lifecycle, daemon election/restart/shutdown, routing HTTP, WebSocket/RPC, auth boundaries, discovery, version guards, embedded/sidecar modes y contratos cliente-servidor.

Compara siempre contra `dev`. Escribe exclusivamente en la branch `reverse-engineering`, bajo `reverse-engineering/analysis/09-backend-transports/`, con subcarpetas por branch o familia demostrablemente equivalente y `README.md` de síntesis. Cita paths, símbolos, commits/PRs, endpoints y separa hechos de inferencias.