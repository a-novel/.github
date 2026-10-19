<!--
Before you submit:
  - Title the pull request as a Conventional Commit: <type>(<scope>): <description>
    Examples: feat(dao): add key revoke endpoint
              fix(handlers): return 404 instead of 500 on missing key
              chore(deps): bump golib to v1.4.2
  - Name the branch with the same vocabulary: <type>/<scope>/<short-description>
  - Read the Code of Conduct: https://github.com/a-novel/.github/blob/master/CODE_OF_CONDUCT.md

Delete this comment block once your pull request is ready.
-->

## Summary

<!-- 1–3 short bullets on what changed and why. Reviewers read the diff for the how. -->

-
-

## Linked issues

<!-- Write "Closes #123" to close the issue on merge, or "Refs #123" to link it without closing.
     An issue in another repository, such as a planning issue in the org's .github repository,
     needs its full reference: "Closes a-novel/.github#123". A bare "#123" only finds an
     issue in this repository, and the board never moves the other one.
     Without a related issue, write "None" and give the motivation. -->

Closes #

## Type of change

<!-- Tick all that apply. -->

- [ ] `feat` — new capability
- [ ] `fix` — bug fix
- [ ] `refactor` — restructuring without behavior change
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
