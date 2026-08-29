# RULE: Commit Message

Create commit messages using Conventional Commit format `type(scope): summary`, except for merge and revert commits explicitly defined below.

## Rules:

- Normal commits MUST use the format `type(scope): summary`
- `type` MUST be one of: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`
- `scope` SHOULD identify the primary affected package or subsystem
- Example scopes include `core`, `tui`, `app`, `desktop`, `sdk`, `plugin`, `provider`, and `console`
- Omit `(scope)` when no single package or subsystem is the primary target of the change
- `type` and `scope` MUST use lowercase
- `summary` MUST describe the concrete change using a short phrase
- `summary` MUST NOT end with a period `.`
- One commit SHOULD contain changes serving one implementation purpose
- Merge commits MAY use the Git-generated format `Merge pull request #<number> from <branch>`
- Revert commits MAY use the Git-generated format `Revert "<original commit summary>"`

## Examples:

- `fix(core): remove dependency from authentication`
- `feat(console): animate usage allowances and bonuses`
- `chore: update dependency hashes`
- `Merge pull request #123 from feature/authentication`
- `Revert "fix(core): remove dependency from authentication"`
