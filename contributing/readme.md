# Contributing

This document is a generic contribution guideline. Please refer to the specific guidelines for your use case as well:

- [Backend](go/readme.md)

## Discuss first

Discuss your contributions with the project maintainers for optimal guidance.

## Creating branches

Once you are ready to work, make sure you are up-to-date with the master branch

```shell
git checkout master
git pull
```

Then create a new branch with a descriptive name

```shell
git checkout -b type/my-feature
```

A branch name starts with a type:

- `feat`: add some new **user-facing** functionality
- `chores`: backend work that has no direct impact on the application (dependencies update, refactor, etc.)
- `security`: security-related fixes and work
- `fix`: bug fixes
- `community`: work related to the community (documentation, etc.)

The type is followed by a short description, that ideally matches the main commit message:

- Make it short (preferably, < 100 characters), while still being descriptive.
- kebab-case (lowercase, words separated by `-`)

## Opening Pull Requests

Once your work on a branch is ready, you may open a Pull Request. This will start the review process, and may be
followed by the merging of your work.
