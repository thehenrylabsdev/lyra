---
status: accepted
date: 2026-11-26
decision-makers: maintainer
---

# Envelope vault with per-OS custody and assurance levels L0–L3

## Context and problem statement

No two OSes offer the same secret-storage primitive. Only macOS binds a secret to one signed app (data-protection
keychain with an access group). Windows and Linux offer per-user stores. Lyra needs one vault design with the same
semantics everywhere, honest labelling of what each OS gives, and plain-language guidance for users.
See [secrets.md](../design/secrets.md).

## Decision drivers

* No plaintext fallback, ever (L-01); no `vault/get` over the protocol.
* Per-consumer scopes that cannot be tampered with.
* The same code path on three keystores of different capability; no silent downgrade.
* Users must understand their protection level and how to raise it.

## Considered options

* One OS keystore item per secret (Keychain, Credential Manager, Secret Service).
* Envelope vault: one VMK in the strongest store per OS, secrets AEAD-encrypted under it.
* Electron `safeStorage`.
* Require per-app L3 on every OS (VBS enclave or a vault service on Windows; Flatpak on Linux).

## Decision outcome

Chosen option: **envelope vault** (`lyra-vault`, about 0.9k LOC Rust). One 32-byte VMK in the strongest store; each
secret XChaCha20-Poly1305-encrypted under an HKDF-derived key with name, kind and `allowedConsumers` in the AAD; the DB
key is `HKDF-SHA256(VMK, "lyra/db/v1")`. Custody and levels:

| OS | Custody | Level |
|---|---|---|
| macOS (signed) | Data-protection keychain, group `TEAMID.dev.thehenrylabs.lyra.vault` | L3 |
| macOS (dev build) | Login keychain item bound to the binary | L1, labelled |
| Windows | TPM-bound CNG key (Platform Crypto Provider); DPAPI only with opt-in | L2; L1 |
| Linux | `systemd-creds --user` + TPM2 (systemd ≥ 256); else Secret Service | L2; L1 |
| None usable | Refuse; session-only mode offered | L0 |

The level is reported by `vault/status`, shown in the UI with the guidance in [secrets.md](../design/secrets.md) §3,
and asserted per CI runner. `minVaultLevel` per secret kind (L1 API keys, L2 OAuth refresh tokens and mTLS certs) can
be lowered only through a native dialog.

### Consequences

* Good, because one hardware-bound root replaces many items and avoids Credential Manager size and concurrency limits.
* Good, because scope tampering fails AEAD authentication.
* Bad, because Windows and Linux secrets are per-user: same-user malware can ask the TPM or Secret Service. Compensated
  by `lyra-harden`, a presence gate, the sandbox denying keystore access, and honest labelling; a VBS-enclave tier is v1.1.
* Bad, because losing the TPM key loses the VMK; recovery is re-entry or an optional passphrase-wrapped backup.
* Bad, because Ubuntu 22.04/24.04 and Debian 12 users get L1 until their distro ships systemd ≥ 256.

### Confirmation

Tests ST-1..ST-6 and A1..A16; spikes S-VAULT-M1, S-VAULT-W1, S-VAULT-W3, S-VAULT-L1..L3, S-VAULT-V2.

## Pros and cons of the options

### One item per secret

* Bad, because semantics differ per OS, Credential Manager loses concurrent writes, and every item is a separate surface.

### `safeStorage`

* Bad, because it is main-process only and falls back to a hardcoded key (`basic_text`) on Linux.

### L3 everywhere

* Bad, because it needs a VBS enclave or a privileged service on Windows in v1 and has no native Linux answer without Flatpak.

## More information

Revisit when the VBS-enclave spike (S-VAULT-W6) passes or Linux gains a per-app secret primitive for native packages.
