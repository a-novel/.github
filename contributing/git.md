# Git Workflow

## Git Conventions

- **Always lowercase** for branch names and commit messages
- `master` is the main branch — create feature branches from it
- One branch per feature/fix — keep changes focused

## Branch Naming

Format: `type/scope/short-description`

```
feat/auth/two-factor-login
fix/dao/credentials-null-check
chore/deps/upgrade-chi
```

**Types:**

| Type    | Usage                                       |
| ------- | ------------------------------------------- |
| `feat`  | New feature or capability                   |
| `fix`   | Bug fix                                     |
| `chore` | Maintenance (dependencies, config, tooling) |
| `test`  | Adding or updating tests                    |
| `perf`  | Performance improvements                    |

**Scopes** describe the affected area:

- **Layer**: `dao`, `services`, `handlers`, `cmd`, `config`
- **Domain**: `token`, `credentials`, `shortcode`, `auth`
- **General**: `deps`, `format`, `tests`, `ci`

## Commit Messages

Format: `type(scope): short description`

```
feat(handlers): allow users to login with shortcodes
fix(dao): handle null email in credentials lookup
chore(deps): upgrade testify to v1.9.0
```

Keep descriptions concise and imperative ("add", "fix", "update" — not "added", "fixes", "updating").

### Hotfixes

`hotfix` is **reserved for direct commits to `master`** in urgent situations. Do not use it for branch names.

```
hotfix(auth): patch token expiration vulnerability
```

Hotfixes bypass the normal PR flow and should be used sparingly for critical production issues only.
