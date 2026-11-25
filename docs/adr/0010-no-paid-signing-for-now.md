---
status: accepted
date: 2026-11-26
decision-makers: maintainer
---

# No paid code signing for now: unsigned or ad-hoc development builds

## Context and problem statement

Several controls depend on code signing: the macOS data-protection keychain (needs a provisioning profile and Team ID),
signature-based peer authentication, TCC grant persistence, notarization, Windows package identity and SmartScreen
reputation. Signing certificates and developer programs cost money, and the project is not buying them yet. How does
each control behave until a production-signing epic lands?

## Decision drivers

* Development must proceed on all three OSes without paid signing.
* No control may silently claim more than it delivers; every weaker mode is labelled.
* Release-only protections must not leak into dev builds as false assurance, and dev-only relaxations must never ship.

## Considered options

* Buy signing now.
* **Unsigned or ad-hoc signed development builds with labelled fallbacks; production signing as a later epic.**
* Self-signed certificates distributed to testers.

## Decision outcome

Chosen option: **unsigned or ad-hoc signed development builds.** macOS builds are ad-hoc signed (`codesign -s -`,
hardened runtime on); Windows and Linux builds are unsigned. The UI shows a persistent "Development build" banner, and
`vault/status` reports `build: "dev"`. Production signing (Developer ID + notarization, Windows Authenticode, GPG
repository keys on hardware tokens) is a later epic.

| Control | Dev build behaviour |
|---|---|
| macOS vault | Data-protection keychain unavailable (no provisioning profile). VMK in a login-keychain item bound to the binary: **L1, labelled "development build"**. Prompts may recur after each rebuild |
| Windows / Linux vault | Unchanged: L2 with TPM (and systemd ≥ 256 on Linux), else L1. Signing does not affect these levels |
| Peer authentication | Falls back to the **pairing nonce** over an inherited fd plus same-user checks (uid / SID, DACL). Labelled "development (nonce only)" |
| macOS TCC | Grants are keyed to the code signature, which changes every ad-hoc build, so **TCC grants reset per build** (`xtask tcc-reset` clears stale entries) |
| Windows sandbox | MXC works unpackaged. If strict proxy mode needs package identity (spike S-W2), dev builds MAY use the identity-less mode, labelled; release builds refuse it |
| Linux sandbox | A build tree is outside `/opt/Lyra`; on Ubuntu 23.10+ `xtask dev-apparmor` installs a profile for the dev bwrap path, else T1 reports `sandbox_unsupported` |
| Electron fuses, ASAR integrity, CSP, renderer sandbox | Same as release (ad-hoc signing still seals the macOS bundle; ASAR integrity is weaker because anyone can re-sign ad-hoc) |
| Hardened core | Windows `BinarySignaturePolicy` (Microsoft + Lyra signed only) is off; ACG, CFG and `NoRemoteImages` stay on |
| Linux launcher hash-list check | Skipped (no root-owned install), labelled |
| Updates | Updater disabled; no update manifest is fetched |
| Gatekeeper, SmartScreen | Locally built apps are not quarantined; downloaded nightlies warn, as expected |

### Consequences

* Good, because all three OSes can be developed and tested now, and every weaker mode is visible.
* Bad, because the macOS L3 vault, signature-based peer checks and persistent TCC grants cannot be tested until the
  signing epic; spikes S-CORE2 (macOS and MSIX parts), S-VAULT-M1, S-VAULT-M2, S-SH1, S-D3 and S-SC1 wait for it.
* Bad, because dev builds must be used with test secrets only; the UI says so.

### Confirmation

Test A2 asserts L1 on macOS dev runners and L3 on signed runners once they exist. A release check fails if any
dev-only relaxation (identity-less MXC mode, skipped hash-list check, dev banner off) is present in a release build.

## More information

Signing certificates cost money; buying them needs the maintainer's explicit approval and opens the
production-signing epic.
