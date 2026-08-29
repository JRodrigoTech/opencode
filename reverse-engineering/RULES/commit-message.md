# RULE: Commit Message

When creating a commit, always use Conventional Commit format `type(scope): summary`

Rules:

- `type` MUST be one of: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`
- `scope` SHOULD identify the primary affected package or subsystem, such as `core`, `tui`, `app`, `desktop`, `sdk`, `plugin`, `provider`, or `console`
- Omit `(scope)` only when the change is genuinely cross-cutting or no meaningful scope exists
- `summary` MUST be short, specific, and describe the concrete change
- Use lowercase for `type` and `scope`
- Do not end the summary with a period `.`
- One commit SHOULD represent one coherent change

Examples:

- `fix(core): remove dependency from authentication`
- `feat(console): animate usage allowances and bonuses`
- `chore: update dependency hashes`
- `Merge pull request #123 from feature/authentication`
- `Revert "fix(core): remove dependency from authentication"`
