<!--
Thanks for opening a pull request against an a-novel repository!

Before you submit, please make sure:
  - The PR title follows Conventional Commits: <type>(<scope>): <description>
    Examples: feat(dao): add key revoke endpoint
              fix(handlers): return 404 instead of 500 on missing key
              chore(deps): bump golib to v1.4.2
  - The branch name follows the same vocabulary: <type>/<scope>/<short-description>
  - You have read the Code of Conduct (../CODE_OF_CONDUCT.md).

You can delete this comment block once your PR is ready.
-->

## Summary

<!-- 1–3 short bullets describing WHAT changed and WHY. Skip the "how" — reviewers will read the diff. -->

-
-

## Linked issues

<!-- Use "Closes #123" to auto-close on merge, or "Refs #123" to link without closing.
     If there is no related issue, write "None" and briefly explain the motivation. -->

Closes #

## Type of change

<!-- Tick all that apply. -->

- [ ] `feat` — new capability
- [ ] `fix` — bug fix
- [ ] `refactor` — restructuring without behaviour change
- [ ] `perf` — performance improvement
- [ ] `test` — tests only
- [ ] `docs` — documentation only
- [ ] `chore` — maintenance, dependencies, build, etc.
- [ ] `ci` — CI/CD pipeline only
- [ ] `revert` — revert of a previous commit

## Breaking changes

<!-- If this PR introduces a breaking change, add `!` to the commit type (e.g. `feat(proto)!: …`)
     AND a `BREAKING CHANGE:` footer in the commit body. Describe the break and the migration
     path below. -->

- [ ] This PR contains **no** breaking changes.
- [ ] This PR contains breaking changes (described below).

<!-- If breaking, describe:
       - what breaks
       - which consumers are affected
       - the migration steps users must take
-->

## Test plan

<!-- How did you verify the change? Tick every check that applies, and add any manual steps
     reviewers should run locally. -->

- [ ] Tests pass locally (`a-novel test -y` / `pnpm test` / equivalent)
- [ ] Integration tests pass locally where applicable
- [ ] Linters and formatters pass (`pnpm lint` / `pnpm lint:go` / equivalent)
- [ ] Manually verified in a local dev environment (describe steps below)

<!-- Manual verification steps:

1.
2.
-->

## Screenshots / recordings

<!-- For UI changes, paste before/after screenshots or a short recording. Otherwise, delete this
     section. -->

## Checklist

- [ ] PR title follows Conventional Commits and is under 70 characters
- [ ] Commits are split into logical units (one DAO file + its test = one commit, etc.)
- [ ] No secrets, credentials, or `.env` files are committed
- [ ] Documentation (READMEs, doc comments, OpenAPI, proto comments) is updated where relevant
- [ ] Generated artifacts (proto bindings, mocks, OpenAPI clients) are committed alongside the
      source change that required them
- [ ] CI is green (or the remaining failures are unrelated and explained below)
