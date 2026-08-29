# RULE: Functional Split

Split growing files by functional responsibility only when the resulting units remain cohesive, independently understandable, and connected through simple one-way dependencies.

## Rules:

- File length alone MUST NOT trigger a split
- A file SHOULD be split when it contains multiple functional units with distinct responsibilities
- A functional unit SHOULD have clear inputs and outputs
- A functional unit SHOULD be independently testable
- Functional units SHOULD NOT depend on shared mutable state
- Splitting MUST NOT introduce circular dependencies
- Splitting SHOULD NOT increase coupling, indirection, callbacks, or state synchronization
- When a feature is split, one module SHOULD own the feature-level flow
- Feature-level orchestration SHOULD occur through the owning module rather than through sibling-to-sibling coordination
- Mutable feature state MUST have one clear owner
- Shared lower-level contracts, types, configuration values, or pure primitives MAY be imported by multiple functional units
- Shared dependencies MUST remain one-way and acyclic
- Cohesive algorithms SHOULD remain together when separating them would make their behavior harder to understand or coordinate
- Managers, interfaces, wrappers, helpers, utility layers, or other abstractions MUST NOT be created only to satisfy this RULE
- Public behavior SHOULD remain unchanged when code is split

## Examples:

Prefer:

```text
feature.py

→

feature/
├── __init__.py
├── main.py
├── unit_a.py
└── unit_b.py
```

Prefer feature-level orchestration through the owner:

```text
main.py -> unit_a.py
main.py -> unit_b.py
```

Avoid sibling orchestration:

```text
unit_a.py <-> unit_b.py
```

## Priority:

- cohesion > file size
- simple dependencies > artificial separation
- one-way flow > cross-module coordination
- clear state ownership > distributed mutable state
- behavioral simplicity > additional abstraction
