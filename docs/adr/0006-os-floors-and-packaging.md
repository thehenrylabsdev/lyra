---
status: accepted
date: 2026-11-26
decision-makers: maintainer
---

# OS floors and packaging: macOS 15+, Windows 11 at the MXC floor, Linux .deb/.rpm only

## Context and problem statement

Lyra ships on Windows, macOS and Linux in v1 with consistent security. Some controls only exist from certain OS
builds (MXC, `SO_PEERPIDFD`, systemd-creds with TPM2), and some package formats cannot install what the sandbox needs.
Which OS versions and package formats does v1 support? See [stack.md](../design/stack.md) §1.

## Decision drivers

* The coding agent is disabled rather than weakened where its sandbox cannot run (no silent downgrade).
* Protected install locations on every OS (app integrity; Linux lacks ASAR integrity).
* Linux packages must be able to install AppArmor profiles for bubblewrap and Chromium.
* Keep the support matrix testable by one maintainer.

## Considered options

* macOS 14+ vs **15+**.
* Windows 10 + 11 with a weaker sandbox fallback vs **Windows 11 at the MXC build floor only**.
* Linux: .deb/.rpm, AppImage, Flatpak, Snap.

## Decision outcome

Chosen options:
- **macOS 15+** (DMG + ZIP). Covers Electron 45, ScreenCaptureKit one-shot capture and `SMAppService`.
- **Windows 11 at the build MXC requires:** 24H2 26100.9278+, 25H2 26200.9278+, 26H2 26300.9550+, 26H1 28000.2804+;
  .9550+ recommended. **No Windows 10.** Packaging: **MSIX**, plus MSI for enterprise per-machine installs. Older
  builds report `sandbox_unsupported` and disable the coding agent.
- **Linux: .deb and .rpm only**, from signed apt/dnf repositories, installing to root-owned `/opt/Lyra` with AppArmor
  profiles. Supported: Ubuntu 22.04/24.04+, Debian 12/13, current Fedora, x64 and arm64.

Not offered in v1: Flatpak (its sandbox blocks the nested user namespaces bubblewrap needs), AppImage (cannot install
AppArmor profiles or a root-owned install), Snap, Windows 10.

### Consequences

* Good, because every supported install has a protected location and the profiles its sandboxes need.
* Good, because there is no weaker Windows sandbox fallback with known fence bypasses.
* Bad, because Windows users below the August 2026 cumulative update cannot use the coding agent until they update;
  the UI shows a clear update prompt.
* Bad, because Linux kernels before 6.5 (Ubuntu 22.04 GA, Debian 12) lack `SO_PEERPIDFD`; named-socket peer checks fall
  back to `SO_PEERCRED`, and privileged channels use inherited socketpairs only.
* Bad, because users of immutable distros that prefer Flatpak are not served in v1.

### Confirmation

CI matrix on the floors; preflight probes (R9) report `sandbox_unsupported` below them; spikes S-W1 (MXC builds),
S-L1 (five distros), S-SH1 (MSIX) and S-SH5 (apt/dnf chain).

## Pros and cons of the options

### Windows 10 support

* Bad, because MXC does not run there, and the alternatives need UAC or have documented fence bypasses.

### Flatpak

* Good, because the Secret portal would give a per-app L3 vault on Linux.
* Bad, because bubblewrap cannot create user namespaces inside the Flatpak sandbox, so sandbox parity cannot be shown.

### AppImage

* Bad, because it is user-writable, cannot install AppArmor profiles, and its signatures are not enforced at runtime.

## More information

Revisit Flatpak if an `--allow-userns` equivalent lands or a Landlock-only mode passes the sandbox kill criteria.
