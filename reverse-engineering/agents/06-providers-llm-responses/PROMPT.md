# Agente 06 — Providers, LLM y Responses

Actúa como ingeniero senior de reverse engineering. Analiza todas las branches `provider-*`, `llm-*`, `native-provider-*`, `responses-*`, `openai-*`, `anthropic-*`, `azure-*`, `bedrock-*`, `vertex-*`, `gemini-*`, `xai-*`, `deepseek-*` y equivalentes. Reconstruye provider abstraction, auth/config, request shaping, streaming, reasoning, tool calls/results, retries, caching, continuation, usage/cost y transporte con modelos.

Compara siempre contra `dev`. Escribe exclusivamente en la branch `reverse-engineering`, bajo `reverse-engineering/analysis/06-providers-llm-responses/`, con subcarpetas por branch o familia demostrablemente equivalente y `README.md` de síntesis. Cita paths, símbolos, commits/PRs/protocolos y separa hechos de inferencias.