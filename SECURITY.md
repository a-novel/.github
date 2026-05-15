# Security Policy

This document outlines the security policy for the [`a-novel`](https://github.com/a-novel)
organization and **applies to every repository under it**, unless an individual repository
publishes its own `SECURITY.md` that supersedes this one.

- [Reporting a Vulnerability](#reporting-a-vulnerability)
- [What to Include in a Report](#what-to-include-in-a-report)
- [Supported Versions](#supported-versions)
- [Scope](#scope)
- [Out of Scope](#out-of-scope)
- [Disclosure Policy](#disclosure-policy)
- [Recognition](#recognition)
- [Comments on This Policy](#comments-on-this-policy)

## Reporting a Vulnerability

The `A-Novel` team takes all security reports seriously. Thank you for helping us keep our users
and contributors safe — we appreciate your time and your responsible disclosure.

**Please do not open public GitHub issues, pull requests, or discussions for security reports.**
Public reports give attackers a head start before a fix is ready.

You have two preferred reporting channels:

1. **GitHub private vulnerability reporting (recommended).** Navigate to the affected repository
   on GitHub, open the **Security** tab, click **Report a vulnerability**, and submit a private
   advisory. This keeps the report scoped to the affected repository and gives us a private
   workspace to coordinate the fix with you.
2. **Email.** Write to the lead maintainer at
   **<geoffroy.vincent@agorastoryverse.com>**. Use a clear subject line that starts with
   `[security]` so the report is not lost in normal traffic.

You should receive an acknowledgement within **48 hours**, and a more detailed response within
**a further 48 hours** indicating the next steps. From there we will keep you informed of progress
toward a fix and may ask follow-up questions.

If the vulnerability lives in a **third-party dependency** rather than our own code, please report
it to that dependency's maintainers and let us know the upstream report so we can track the fix.

## What to Include in a Report

A useful report typically contains:

- The affected repository (and, if known, the affected version, commit SHA, or release tag).
- A description of the vulnerability and the impact you believe it has.
- Reproduction steps, a proof-of-concept, or a minimal example.
- Any relevant logs, screenshots, or network traces — **with any secrets redacted**.
- Your assessment of severity (CVSS or informal) if you have one.
- Whether you would like to be credited publicly, and under what name.

Do not perform testing that could degrade service for other users (denial-of-service, mass account
creation, data destruction). If you are unsure whether a test is safe, **ask first**.

## Supported Versions

Unless an individual repository's `SECURITY.md` says otherwise, only the **latest released
version** of each package or service is supported with security fixes. For services and
applications, "latest released" means the current `master` branch and the most recent published
container image or deployment. For libraries, it means the most recent Git tag on the default
branch.

Older versions may receive a fix at the maintainers' discretion when the fix is trivial to
backport, but this is not guaranteed. If you depend on an older version in production, please
plan to upgrade.

## Scope

The following are in scope for this policy:

- All source code in repositories under the [`a-novel`](https://github.com/a-novel) and
  [`a-novel-kit`](https://github.com/a-novel-kit) GitHub organizations.
- Published artifacts produced by those repositories — Go modules, npm packages, container
  images, and any other release artifact distributed under either organization.
- Infrastructure that is owned and operated by the `A-Novel` team and serves project users
  directly.

## Out of Scope

The following are **out of scope** for this policy. Please do not file security reports for them
here — report them to the responsible party instead.

- Vulnerabilities in third-party services that we use but do not operate (e.g., GitHub itself,
  cloud providers, SaaS dependencies).
- Vulnerabilities in third-party libraries that we depend on — report those to the library's
  maintainers. We will track and apply upstream fixes once they are released.
- Social-engineering attacks targeting individual contributors, employees, or community members.
- Findings from automated scanners with no demonstrated impact, missing best-practice headers
  with no exploit path, or theoretical issues without a working proof-of-concept.

## Disclosure Policy

When a valid security report is received, a primary handler will be assigned to coordinate the
response. The handler will:

1. Confirm the problem and determine the set of affected versions and repositories.
2. Audit related code for similar issues that may share the root cause.
3. Prepare fixes for all releases still under support and merge them privately.
4. Publish a coordinated release containing the fix.
5. Publish a public security advisory (and, where applicable, a CVE) crediting the reporter.

We aim to release a fix within **90 days** of the initial report for most issues, and faster for
critical ones. We will coordinate the public disclosure date with the reporter and try not to
disclose until a fix is available.

## Recognition

If you would like to be credited in the public advisory, tell us in the initial report and let us
know the name (and optional link) you would like us to use. Anonymous reporters are equally
welcome.

## Comments on This Policy

If you have suggestions on how this policy could be improved, please open a pull request against
this file in the [`.github`](https://github.com/a-novel/.github) repository.
