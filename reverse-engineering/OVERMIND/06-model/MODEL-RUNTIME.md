# Model Runtime Lessons

## Preserve Overmind ModelService

Overmind ya tiene un boundary fuerte:

```text
ModelService
 -> GenerationExecutor
 -> ModelTarget
 -> ModelBackend
```

`GenerationExecutor` diferencia technical retry y semantic recovery. `ModelService.generate_internal` limita tools y budgets para cognition interna. Mantenerlo.

## Qué extraer de OpenCode

### Normalized event vocabulary

OpenCode logra que AI SDK y native runtime terminen en `LLMEvent`. Overmind tiene callbacks + normalized generation result. Para EventPort/UI/background conviene formalizar un vocabulary interno independiente del backend:

```text
ModelEvent
- activity
- thinking_start/delta/end
- content_start/delta/end
- tool_call_complete
- usage/final
- provider_failure
```

No todos son public Runtime Events; un adapter puede convertir ModelEvent -> SIGNAL/FACT.

### Provider adapters isolate wire quirks

Cuando se añada otro backend, mantener toda autenticación/payload/SSE/tool-call assembly/provider error mapping en `models/backends/<id>/`, como OpenRouter hace hoy.

### ModelCapabilities over model-name heuristics

Añadir features por declared capabilities, no por `if "gpt" in model_id`.

Capabilities futuras:

- tools;
- streaming;
- textual thinking;
- structured output;
- context/output maxima;
- continuation support;
- image/file input;
- prompt caching if semantics matter.

## Multi-target routing

Overmind ya permite named targets por mapping. La evolución recomendada es:

1. explicit caller-selected `target_id`;
2. subagent spec target;
3. optional deterministic policy router;
4. cognitive/automatic routing solo si una necesidad medida lo justifica.

No introducir provider registry o fallback graph global antes de estos pasos.

## Usage/cost

Overmind ya conserva Usage por physical response y aggregates logical generation. Si aparecen priced backends, añadir cost como normalized accounting derivado de authoritative usage/model pricing, no mezclarlo con Context token prediction.
