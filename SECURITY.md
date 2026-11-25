# Security policy

Lyra handles API keys, OAuth tokens, browser sessions and the ability to run code on your machine. We take reports
seriously.

## Reporting a vulnerability

**Do not open a public issue.** Report privately through GitHub:
**Security → Report a vulnerability** on this repository
([private vulnerability reporting](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability)).

Please include:
- the affected component (vault, sandbox, egress proxy, approvals, browser, computer use, MCP, updater);
- the OS and version;
- reproduction steps;
- the impact.

We aim to acknowledge reports within 3 working days and to agree a disclosure date with you.

## Supported versions

Lyra is pre-alpha. Only `main` is supported until the first release.

## Scope

In scope:
- **Escapes and bypasses:**
  - sandbox escapes;
  - egress-policy bypasses;
  - approvals forged from the renderer;
  - computer-use actions outside allowed apps.
- **Secret exposure:**
  - a secret leaking to the model, logs, disk or the sandbox;
  - vault access by an unauthorized process beyond the disclosed assurance level.
- **Supply-chain issues in our build and release.**

Weaker protections that an OS is documented to have are out of scope. One example is that Windows and Linux key stores
are per-user, not per-app. See [docs/design/secrets.md](docs/design/secrets.md). Ways to break those documented limits
are in scope.
