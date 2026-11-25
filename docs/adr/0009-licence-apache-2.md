---
status: accepted
date: 2026-11-26
decision-makers: maintainer
---

# Licence: Apache-2.0

## Context and problem statement

Lyra is open source. It copies code from Apache-2.0 projects ([`openai/codex`](https://github.com/openai/codex) SBPL
fragments and seccomp design; [`anthropic-experimental/sandbox-runtime`](https://github.com/anthropic-experimental/sandbox-runtime)
mach-lookup allowlist and denied ranges), depends on MIT and Apache-2.0 crates and packages, and ships bubblewrap
(LGPL-2.0-or-later) as a separate binary. Which licence should Lyra's own code use?

## Decision drivers

* Compatibility with code copied from Apache-2.0 projects, with notices kept.
* An explicit patent grant, given a security-sensitive codebase that others may embed.
* Contributor and downstream clarity; no copyleft obligations on users of the app.
* Compatibility with every runtime dependency in [stack.md](../design/stack.md).

## Considered options

* Apache-2.0.
* MIT.
* MPL-2.0.
* AGPL-3.0.
* Proprietary.

## Decision outcome

Chosen option: **Apache-2.0** for all of Lyra's own code and documentation.

Obligations this creates:
- A top-level `LICENSE` (Apache-2.0) and `NOTICE`. `NOTICE` keeps the attribution and notices of copied Apache-2.0 code
  (Codex, sandbox-runtime), and each copied file keeps its original header plus a "modified by Lyra" note.
- bubblewrap ships as a separate, unmodified binary with its LGPL licence text and a written source offer; it is never
  linked into Lyra code.
- MIT dependencies (for example `mxc-sdk`) keep their notices in the third-party licence bundle generated at release.
- Contributions are accepted under Apache-2.0 (inbound = outbound, section 5 of the licence).
- No dependency under a non-OSI licence or AGPL ships in the app (Phoenix ELv2, Mastra `ee/`, AGPL perception modules
  are excluded); `cargo-deny` and the JS licence check enforce an allowlist.

### Consequences

* Good, because it matches the copied code's licence and gives users a patent grant.
* Good, because companies can adopt and embed Lyra without copyleft concerns.
* Bad, because others may ship closed derivatives; accepted.
* Bad, because NOTICE and third-party bundles need maintenance; generated in CI from `cargo-about`-style metadata and
  the SBOM.

### Confirmation

CI licence gate (`cargo-deny` licences, JS licence allowlist) and a release check that `NOTICE`, the LGPL source offer
and the third-party bundle are present.

## Pros and cons of the options

### MIT

* Good, because it is short and permissive.
* Bad, because it has no explicit patent grant, and the copied Apache-2.0 code keeps its own terms anyway.

### MPL-2.0

* Bad, because file-level copyleft adds obligations without a clear benefit for a desktop app.

### AGPL-3.0

* Bad, because it deters adoption and complicates a later self-hosted server mode for users.

### Proprietary

* Bad, because it conflicts with the project's purpose and with community contribution.
