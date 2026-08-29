# RULE: Rule Creation

Create each RULE as a minimal, deterministic operational specification for an LLM.

Rules:

- Each RULE MUST govern one behavior, constraint, or decision domain
- Use the title format `# RULE: <Name>`
- State the primary expected behavior immediately after the title
- Use `MUST`, `MUST NOT`, `SHOULD`, `SHOULD NOT`, and `MAY` to express requirement strength
- Instructions MUST describe observable actions, conditions, constraints, or decisions
- Trigger conditions MUST be explicit when the RULE is conditional
- Exceptions MUST be explicit when they alter normal behavior
- Instructions within the same RULE MUST NOT contradict each other
- Instructions MUST NOT be duplicated or restated with equivalent meaning
- Ambiguous terms MUST NOT be used unless explicitly defined by deterministic conditions
- Content unrelated to the RULE's decision domain MUST NOT be included
- Examples SHOULD be minimal and included only when they reduce ambiguity
- Structural or syntactic requirements SHOULD use code blocks when representation matters
- Add `Priority` only when the RULE contains principles that can conflict
- Conflicting principles MUST be ordered from highest to lowest priority
- Optional sections MUST be omitted when they provide no additional decision information
- The RULE SHOULD remain independently understandable
- Remove any text that does not change how the LLM behaves or decides

Preferred structure:

```text
# RULE: <Name>

<Primary instruction>

Rules:

- <requirement>
- <constraint>
- <exception>

Examples:

- <example>

Priority:

- <principle A> > <principle B>
```

Priority:

- internal consistency > instruction quantity
- deterministic behavior > explanatory prose
- explicit conditions > implicit assumptions
- single responsibility > broad policy
- minimal specification > redundant content
