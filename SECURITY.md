# Security Policy

This policy covers every repository of the [`a-novel`](https://github.com/a-novel) organization.
A repository that publishes its own `SECURITY.md` replaces it.

## Reporting a Vulnerability

Report a vulnerability privately. Never open a public issue, pull request or discussion for it. A public
report gives attackers a head start before a fix is ready.

Use one of two channels:

1. **GitHub private vulnerability reporting**, preferred. Open the affected repository's **Security** tab,
   click **Report a vulnerability** and submit the advisory. The report stays scoped to that repository,
   and the advisory gives us a private place to work on the fix with you.
2. **Email** the lead maintainer at **<geoffroy.vincent@agorastoryverse.com>**. Start the subject line
   with `[security]`.

We acknowledge a report within **48 hours** and send the next steps within **48 more hours**. We keep you
informed until the fix ships and may ask follow-up questions.

When the vulnerability lives in a **third-party dependency**, report it to that dependency's
maintainers, then send us the link to the upstream report so we can track the fix.

## What to Include in a Report

- The affected repository and, if known, the version, commit SHA or release tag.
- The vulnerability and the impact you believe it has.
- Reproduction steps, a proof of concept or a minimal example.
- Relevant logs, screenshots or network traces, **with every secret redacted**.
- Your severity assessment, as a CVSS score or in your own words.
- Whether you want public credit, and under what name.

Never test in a way that degrades the service for other users, such as denial of service, mass account
creation or data destruction. **Ask first** when you are unsure a test is safe.

## Supported Versions

Security fixes target the **latest release** of each package or service, unless a repository's own
`SECURITY.md` says otherwise. For a service or an application, that is the current `master` branch and
the most recent published container image or deployment. For a library, it is the most recent tag on
the default branch.

An older version may get a fix when it is trivial to backport, at the maintainers' discretion. Plan to
upgrade when you run an older version in production.

## Scope

This policy covers:

- All source code in the organization's repositories.
- The artifacts those repositories publish, such as Go modules, npm packages and container images.
- The infrastructure the A-Novel team owns and operates to serve its users.

## Out of Scope

Report these to the party responsible for them:

- Vulnerabilities in third-party services we use but do not operate, such as GitHub or a cloud
  provider.
- Vulnerabilities in third-party libraries we depend on. We apply the upstream fix once it is released.
- Social engineering against contributors, employees or community members.
- Scanner findings with no demonstrated impact, missing best-practice headers with no exploit path, and
  theoretical issues without a working proof of concept.

## Disclosure Policy

A valid report gets a primary handler, who coordinates the response:

1. Confirm the problem and find the affected versions and repositories.
2. Audit related code for issues that share the root cause.
3. Prepare fixes for every supported release and merge them privately.
4. Publish a coordinated release with the fix.
5. Publish a security advisory, and a CVE where applicable, crediting the reporter.

We aim to release a fix within **90 days** of the report, sooner for critical issues. We agree on the
disclosure date with the reporter and avoid disclosing before a fix is available.

## Recognition

To be credited in the advisory, say so in your report, with the name and optional link to use. Anonymous
reports are equally welcome.

## Comments on This Policy

Suggest an improvement in a [discussion](https://github.com/a-novel/.github/discussions) on the
`.github` repository.
