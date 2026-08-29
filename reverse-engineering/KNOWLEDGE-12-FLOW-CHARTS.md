# 12 — Flow charts de bolsillo

Este archivo es una chuleta visual. Cada diagrama elimina detalles para conservar sólo el recorrido esencial.

## A. Un turno normal

```mermaid
flowchart LR
    U[User input] --> S[Session]
    S --> C[Build context]
    C --> L[LLM stream]
    L --> P[Processor]
    P --> T{Tools?}
    T -- sí --> X[Execute tools]
    X --> S
    T -- no --> F{Continue/compact/retry?}
    F -- sí --> S
    F -- no --> I[Idle]
```

## B. Tool call con permiso

```mermaid
flowchart LR
    L[LLM] --> T[Tool call]
    T --> V[Validate]
    V --> P[Permission]
    P -->|allow| X[Execute]
    P -->|ask| H[Human decision]
    H -->|allow| X
    P -->|deny| E[Error]
    H -->|reject| E
    X --> R[Tool result]
    R --> L
```

## C. Subagente

```mermaid
flowchart LR
    P[Parent Session] --> TASK[task]
    TASK --> C[Child Session]
    C --> A[Subagent profile]
    A --> RUN[Normal Session runtime]
    RUN --> OUT[Result + task_id]
    OUT --> P
```

## D. Skill lazy

```mermaid
flowchart LR
    D[Discover skills] --> C[Catalog names/descriptions]
    C --> L[LLM]
    L --> S[skill(name)]
    S --> P[Permission]
    P --> F[Full SKILL.md]
    F --> CTX[Conversation context]
```

## E. Provider nativo

```mermaid
flowchart LR
    R[LLMRequest] --> RT[Route]
    RT --> PR[Protocol]
    PR --> TR[Transport]
    TR --> API[Provider]
    API --> PR
    PR --> E[LLMEvent]
    E --> S[Session]
```

## F. Evento durable y proyección

```mermaid
flowchart LR
    R[Runtime fact] --> E[EventV2]
    E --> LOG[Durable log seq]
    E --> P[Projector]
    P --> DB[Read model]
    E --> B[Bridge/live publication]
    DB --> C[Client]
    B --> C
```

## G. MCP

```mermaid
flowchart LR
    O[OpenCode MCP Service] --> C[Connect]
    C --> S[Server MCP]
    S --> CAT[Tools/prompts/resources]
    CAT --> R[Normalized catalog]
    R --> T[Tool runtime / API]
```

## H. ACP

```mermaid
flowchart LR
    IDE[IDE ACP] --> A[ACP adapter]
    A --> S[OpenCode Session]
    S --> E[OpenCode event stream]
    E --> A
    A --> IDE
```

## I. TUI sin red

```mermaid
flowchart LR
    T[TUI] --> RPC[Worker RPC]
    RPC --> F[app.fetch]
    F --> API[Backend API]
    API --> RPC
    RPC --> T
```

## J. Desktop

```mermaid
flowchart LR
    M[Electron main] --> S[Spawn sidecar]
    S --> R[Ready URL + auth]
    R --> UI[Renderer]
    UI --> API[HTTP/SSE API]
    API --> S
```

## K. Dependency graph

```mermaid
flowchart TD
    G[Global graph] --> MAP[LocationServiceMap]
    MAP --> L[Location graph]
    L --> FS[Filesystem]
    L --> T[Tools]
    L --> A[Agents/config]
    L --> R[Session/runtime services]
```

## L. Migración strangler

```mermaid
flowchart LR
    OLD[Old path] --> PROD[Product]
    OLD --> BR[Bridge]
    BR --> NEW[New package/service]
    NEW --> PROD
```

Para el porqué de cada flecha, usa el archivo `KNOWLEDGE-*` correspondiente.