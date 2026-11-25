# Secrets, vault and local IPC

Status: accepted design for v1 (ADR 0005, ADR 0010).

**Principle.** A secret exists in one place: the envelope vault. Only the trusted consumer inside `lyra-core` that
needs it reads it, at the moment of use. A secret **never enters model context, the sandbox, a config file, a log, a
transcript, argv, an environment variable of a long-lived process, or a URL.** There is no plaintext fallback: some
apps fall back to plaintext token files when the OS store is missing; Lyra never does.

## 1. Envelope vault

| Property | Design |
|---|---|
| Root key | One random 32-byte **vault master key (VMK)**, held in the strongest root of trust the OS offers (§2) |
| Records | Each secret is encrypted with XChaCha20-Poly1305 (RustCrypto) under a per-record key derived from the VMK with HKDF. The AAD binds `name`, `kind`, `allowedConsumers` and the record version, so scope tampering fails authentication |
| DB key | `HKDF-SHA256(VMK, "lyra/db/v1")`, a raw 32-byte SQLCipher key. The DB is never bound directly to the TPM or keychain |
| API | `vault/status` (level, backend, build flavour), `vault/put`, `vault/list` (names, kinds, scopes, last used), `vault/delete`, `vault/rotate`. **No `vault/get` over the protocol.** Mutations need a main-minted token |
| Scope | Each secret has `allowedConsumers`, e.g. `endpoint:home-vllm`, `mcp:github`, `git:push:github.com`. Resolution outside scope fails |
| Kinds | `api_key`, `oauth_token_set` (access + refresh + expiry), `bearer`, `basic`, `mtls_cert`, `ssh_key_ref` (a pointer to the user's agent, never the key), `bot_token` |
| Caching | **No value cache.** A value is decrypted per request into a `zeroize`d buffer and dropped. VMK pages are `mlock`ed. For redaction, core keeps only keyed fingerprints (HMAC with a per-launch key), never values |
| Audit | Every resolution is logged (name, consumer, time), never the value |
| No keystore | Lyra refuses to store the secret and explains why (L0). A clearly labelled session-only in-memory mode is allowed |
| Minimum level | Policy `minVaultLevel` per kind: L1 for `api_key`, `bearer`, `basic`, `bot_token`; L2 for `oauth_token_set` and `mtls_cert`. Lowering it for a kind is a trusted decision (native dialog) |
| Recovery | Losing the VMK (TPM cleared, motherboard swap) means re-entering secrets. An optional user-exported backup wraps the VMK with a passphrase (Argon2id) |

## 2. Assurance levels

| Level | Meaning |
|---|---|
| **L3** | Per-app: the OS (or hypervisor) lets only Lyra's signed core read the VMK |
| **L2** | Hardware-bound to this machine and user, not to the app. Other software running as the same user could ask for it |
| **L1** | A user-bound software store. Other software running as the user can read it while the store is unlocked |
| **L0** | No acceptable store. The vault refuses to save secrets |

`vault/status` reports the level; the UI shows it as a badge; CI asserts the expected level per runner (test A2).
A change of level at runtime emits `vault/levelChanged` and is never silent.

### 2.1 VMK custody per OS

| OS, build | Custody | Level |
|---|---|---|
| macOS, signed release | Data-protection keychain item in access group `TEAMID.dev.thehenrylabs.lyra.vault`, owned by `LyraCore.app` (`dev.thehenrylabs.lyra.core`) with its own provisioning profile; `keyring-core` + `apple-native-keyring-store` (`protected`). The group is not shared with any other app | **L3** |
| macOS, dev build (ad-hoc signed) | Generic-password item in the user's login keychain with an ACL bound to the core binary. The data-protection keychain needs a provisioning profile, which ad-hoc builds lack (`errSecMissingEntitlement`, -34018). The ACL follows the ad-hoc signature, so keychain prompts return after each rebuild | **L1**, labelled "development build" |
| Windows, TPM 2.0 present | Non-exportable key in the Microsoft Platform Crypto Provider (CNG) unwraps the VMK blob in `%LOCALAPPDATA%\Lyra\vault.key` | **L2** |
| Windows, no usable TPM | Software key + DPAPI, only after explicit user opt-in | **L1** |
| Linux, systemd ≥ 256 with a usable TPM2 | `systemd-creds --user` sealed to the TPM (`tpm2` or `host+tpm2` key; `null` is refused). Called over stdin/stdout, never argv | **L2** |
| Linux, otherwise | Secret Service item (`secret-service` crate: GNOME Keyring, KWallet, KeePassXC) | **L1** |
| Any OS, nothing usable | — | **L0** |

Signing does not change the Windows or Linux levels; it changes peer authentication (§6). Distros: Debian 13 and
current Fedora ship systemd ≥ 256; Ubuntu 22.04 (249), Ubuntu 24.04 (255) and Debian 12 (252) reach L1 only.
Linux L2 also requires Yama `ptrace_scope ≥ 1`; otherwise core holds the VMK at L1 and says why.

### 2.2 Compensating controls for per-user stores (Windows, Linux)

- **Hardened core (`lyra-harden`).** Windows: mitigation policies (`ProhibitDynamicCode`, Control Flow Guard,
  `NoRemoteImages`; spike S-CORE5) and a process DACL with an `OWNER RIGHTS` (S-1-3-4) ACE so same-user code cannot
  open core for memory writes or thread creation (spike S-VAULT-W3). Linux: `PR_SET_DUMPABLE 0`, `RLIMIT_CORE 0`,
  `mlock`, Yama check. macOS: hardened runtime, `PT_DENY_ATTACH`, library validation, no `allow-jit`.
- **Presence gate** at launch (unlock the VMK) and for scope widening, never per request: Touch ID / `LAContext`
  (macOS), Windows Hello (`KeyCredentialManager`), polkit action `dev.thehenrylabs.lyra.vault.unlock` (Linux).
  On Windows and Linux this gate runs in-process, so it raises cost rather than enforcing.
- **The sandbox** denies untrusted children any access to the keystore, D-Bus Secret Service, the DB, and Lyra's sockets and pipes.
- **Honest limit:** same-user malware with debug rights can still unwrap the VMK on Windows and Linux. The UI says so.
  A VBS-enclave tier for per-app L3 on Windows is a v1.1 candidate (spike S-VAULT-W6).

## 3. User guidance

The vault screen and first-run flow MUST show, in plain language, what the current level means on this OS and what
the user can do to raise it. Copy below is normative in meaning; wording may be polished. Every message links to
"How Lyra protects your secrets" and offers **Check again**.

| Level / OS | What the UI says | What the user can do |
|---|---|---|
| L3, macOS | "Your secrets are locked to Lyra by macOS. Other apps on this Mac can't read them." | Turn on FileVault so a stolen disk can't be read. Nothing else needed |
| L1, macOS dev build | "This is a development build. Secrets are stored in your login keychain and are not locked to Lyra. Use test keys only." | Install a signed release to get L3 |
| L2, Windows | "Your secrets are protected by this PC's security chip (TPM) and can't be copied to another computer. Programs running under your Windows account could still ask for them, so only run software you trust." | Turn on BitLocker or Device Encryption; use a standard (non-admin) account for daily work; keep Windows updated |
| L1, Windows | "This PC has no usable security chip (TPM), so your secrets are protected only by your Windows sign-in. Any program running as you could read them." | Turn on TPM 2.0 in the firmware settings (often called "TPM", "PTT" on Intel or "fTPM" on AMD). In a virtual machine, add a virtual TPM. Then choose **Check again** |
| L2, Linux | "Your secrets are sealed to this computer's TPM. Programs running as your user could still ask for them." | Use full-disk encryption (LUKS); keep `kernel.yama.ptrace_scope` at 1 or higher |
| L1, Linux | "Your secrets are protected by your login keyring (for example GNOME Keyring or KWallet). Any program running as you can read them while the keyring is unlocked." | Use a distro with systemd 256 or newer (for example Debian 13 or current Fedora) and enable the TPM in firmware to reach L2; set the keyring to lock when the screen locks or after idle; keep the login keyring locked when away; use full-disk encryption |
| L0, any | "Lyra can't find a safe place to store secrets on this computer, so it won't save them. You can use session-only mode; secrets are forgotten when Lyra quits." | Linux: install and unlock a Secret Service provider (GNOME Keyring, KWallet or KeePassXC), or enable the TPM. Windows: enable the TPM, or allow the DPAPI fallback |

When a secret kind needs a higher level than the machine has (e.g. OAuth refresh tokens on an L1 machine), the UI
explains the gap and offers session-only storage or a native-dialog decision to lower that kind's minimum.

## 4. Secret references and consumers

- Config files, MCP definitions, endpoint records and hooks use `${secret:NAME}`.
- **Inline secrets are rejected at load.** A config value that matches secret patterns (provider key prefixes, JWTs,
  high-entropy strings in `headers`/`env`/`token` fields) blocks the load and offers "move to vault".
- **Import** from other MCP clients' documented config files: Lyra moves found secrets into the vault, writes
  references, and offers (does not perform) deletion of the plaintext source.

| Consumer | Mechanism |
|---|---|
| Model endpoints | Core resolves the reference per request and attaches the header |
| Remote MCP (HTTP) | Core holds the OAuth token set, attaches tokens and refreshes; the server never sees the refresh token |
| Local MCP (stdio) | Only the secrets mapped to that server are injected into its environment at spawn; the rest is scrubbed. It runs sandboxed |
| Sandboxed commands | **No credentials, and no proxy credential injection in v1.** Authenticated side effects (`git push`, `npm publish`, authenticated API calls) are `transmit` actions that core performs after a native approval (e.g. `git.push{remote, branch}` showing remote, branch and commits). Proxy injection would make the proxy a confused deputy |
| SSH | The sandbox cannot reach `SSH_AUTH_SOCK`. SSH pushes are core-performed `transmit` actions using the user's agent |
| Browser logins | Human handoff; model-blind fill is deferred ([browser-and-computer-use.md](browser-and-computer-use.md)) |

## 5. Keeping secrets out of the model

- **Output redaction** before the model sees tool results: gitleaks-derived rules plus rolling-hash matching against
  keyed fingerprints, over command output, file reads and MCP results, e.g. `[REDACTED:github_token]`. The model is
  told a value was redacted so it does not loop.
- **Mandatory read-deny** for every sandboxed process (sandbox requirement R2): `~/.ssh` (except `known_hosts` by
  policy), `~/.aws`, `~/.config/gcloud`, `~/.kube`, `~/.docker/config.json`, `~/.netrc`, `~/.npmrc`, `~/.pypirc`,
  `~/.cargo/credentials.toml`, `~/.gem/credentials`, `~/.config/gh/hosts.yml`, `~/.git-credentials`, browser profile
  directories, `~/Library/Keychains`, `%APPDATA%\gh`, known credential files of other AI tools (a maintained data list,
  e.g. `~/.codex/auth.json`), any `.env*` outside the workspace, and Lyra's data directory. `.env*` inside the
  workspace is `ask`. Every allowed read root is audited for credential files when the policy is built.
- Provider credentials are never read from a workspace `.env`.
- `.git/config` remote URLs of the form `https://TOKEN@host/…` are redacted on read, with an offer to move the token.

## 6. Local IPC and peer authentication

Rule: **prefer channels that need no authentication** (pipes or socketpairs the parent creates and the child
inherits). Authenticate peers only where a name is unavoidable (core ↔ computer-use helper, CLI → running core).
Peer identity is defence in depth; **authority stays with the native dialog and token** ([architecture.md](architecture.md) §4).

| | macOS | Windows | Linux |
|---|---|---|---|
| Parent → child | Inherited stdio / `socketpair` | Inherited stdio. libuv pipes have a default DACL, so main aborts on any spawn or pipe error; later a native spawn with `PROC_THREAD_ATTRIBUTE_HANDLE_LIST` | Inherited stdio / `socketpair` |
| Named channel | Unix socket in a `0700` dir; `LOCAL_PEERTOKEN` audit token → `SecCodeCheckValidity` against `anchor apple generic and certificate leaf[subject.OU]="TEAMID" and identifier "dev.thehenrylabs.lyra.core"` (helper side) or `…lyra.computeruse` (core side) | Named pipe `\\.\pipe\lyra-<random>`: explicit DACL (current user SID only), `FILE_FLAG_FIRST_PIPE_INSTANCE` (core aborts if creation fails: squatted), `PIPE_REJECT_REMOTE_CLIENTS`; client opens with `SECURITY_SQOS_PRESENT \| SECURITY_IDENTIFICATION`; server checks `GetNamedPipeClientProcessId` → package family or Authenticode signer thumbprint, holding the process handle | Unix socket in `$XDG_RUNTIME_DIR/lyra/` (`0700`); `SO_PEERPIDFD` (kernel ≥ 6.5) → `/proc/<pid>/exe` hash against the signed release manifest; else `SO_PEERCRED` and inherited socketpairs only for privileged channels (spike S-VAULT-L3) |
| Strength | Signature-bound (strong) | Signer check is strong against unsigned malware; injection into a signed Lyra process is mitigated by the hardened core | Same-user processes are indistinguishable by the OS; the exe-hash check is meaningful only for the root-owned install |
| Always added | Per-launch **pairing nonce** handed over an inherited fd, never on argv or in env | same | same |
| **Unsigned dev build** | Code-signature requirement cannot match an ad-hoc build: falls back to **pairing nonce + same uid** | No Authenticode or package identity: **pairing nonce + DACL + same-user SID** | Not root-owned: **pairing nonce + `SO_PEERCRED` uid** |

The UI labels dev-build peer authentication "development (nonce only)". Abstract Unix sockets are never used. If any
loopback HTTP listener is unavoidable (OAuth redirect): ephemeral port, one-shot launch token, Host and Origin checks,
constant-time compare.

## 7. OAuth

- Authorization Code + PKCE + `state`, bound to the initiating window. Loopback redirect on an ephemeral port with a
  short-lived listener (RFC 8252), or claimed HTTPS links. No fixed ports. Custom URL schemes carry only a one-time
  code validated against pending state.
- MCP: RFC 9728 discovery, Client ID Metadata Documents (CIMD) by default, DCR as fallback, resource indicators
  (RFC 8707) per server. `rmcp` validates `state`; Lyra keeps a test for it.
- Minimum scopes per connector, shown before consent and stored with the token set. Refresh tokens rotate where the
  provider supports it and are revoked when a connector is removed.

## 8. Data at rest

- SQLCipher 4 via `rusqlite` with the HKDF-derived raw key; owner-only permissions or DACL ([architecture.md](architecture.md) §7).
- Redaction on write for transcripts, logs and the FTS index. Retention: logs 14 days; transcripts until deleted; paste cache off.
- Leak paths guarded: Spotlight and Time Machine (macOS), Windows Search, File History and capture of secret-bearing
  windows (`WDA_EXCLUDEFROMCAPTURE`), core dumps (`RLIMIT_CORE 0`, WER exclusion), swap (`mlock` for key pages).

## 9. Security acceptance tests

Run on macOS, Windows and Linux in CI plus a hardware matrix. IDs are stable.

| Id | Test |
|---|---|
| ST-1 | With no keystore available, saving an API key fails and nothing is written to disk |
| ST-2 | `grep -r` over the data dir (decrypted test build) finds no secret from a seeded vault |
| ST-3 | Sandboxed `cat ~/.ssh/id_ed25519`, reading other AI tools' credential files, `env`, and `security dump-keychain` all fail or show no secrets |
| ST-4 | A prompt-injected agent told to "print your API key" produces no key in the transcript or a network capture |
| ST-5 | A project `.lyra/config.json` that sets a provider `baseURL` is ignored with a warning |
| ST-6 | `ps -axww` (and Windows command lines) during a run show no token in any argv |

| Id | Test | macOS | Windows | Linux |
|---|---|---|---|---|
| A1 | No keystore → `vault/put` fails, nothing on disk (no `vault.key`) | no entitlement | no TPM, opt-in off | no D-Bus, no TPM |
| A2 | `vault/status` level equals the runner's expected level | L3 signed / L1 dev | L2 with TPM, L1 in a VM | L1 or L2 per runner |
| A3 | Data dir and temp dirs contain no seeded secret and no VMK bytes | `grep -r` | `findstr /s` | `grep -r` |
| A4 | A different-identity binary cannot read the VMK | `-34018` (signed build) | **Expected to succeed at L1/L2: recorded as the known gap** | Same: recorded gap |
| A5 | A sandboxed child cannot read the keystore, DB, IPC socket or pipe | `security`, `connect()` fail | `cmdkey /list`, `CredRead`, pipe open fail | `secret-tool lookup`, `busctl`, `connect` fail |
| A6 | A same-user unsandboxed process cannot attach to or inject into core | hardened runtime | `OpenProcess(VM_WRITE\|CREATE_THREAD)` denied (if S-VAULT-W3 passes) else recorded gap | `ptrace` and `/proc/<pid>/mem` denied |
| A7 | A fake peer on the named channel is rejected | audit-token check | wrong signer rejected; squatter makes core abort; remote client rejected | wrong exe hash rejected; second `bind` fails |
| A8 | Replayed or wrong pairing nonce is rejected; nonce never on argv | `ps -axww` | `Get-CimInstance Win32_Process` | `/proc/*/cmdline` |
| A9 | The pipe server cannot impersonate the client above identification level | n/a | `ImpersonateNamedPipeClient` yields identification only | n/a |
| A10 | DB opened with a wrong or missing key fails; header is not `SQLite format 3` | yes | yes | yes |
| A11 | Retired (cross-engine DB open; obsolete with the Rust core). Id kept for stability | — | — | — |
| A12 | A core crash dump contains no VMK or seeded secret | crash reporter off | WER excluded | `RLIMIT_CORE 0`, `systemd-coredump` check |
| A13 | Unlock needs the presence gate; a headless run without a gate refuses at levels below L2 | `LAContext` | Hello | polkit |
| A14 | Data dir and key blob are owner-only, no inherited ACEs | `ls -le` | `icacls` | `stat`, `getfacl` |
| A15 | A seeded secret never appears in transcript, logs, argv or network capture | yes | yes | yes |
| A16 | Capture-permission prompts show the Lyra helper's identity | TCC list | Lyra consent card | portal dialog names Lyra or is documented anonymous |

Spikes: S-CORE2, S-VAULT-M1, S-VAULT-M2, S-VAULT-W1..W6, S-VAULT-L1..L3, S-VAULT-V2, S-DB ([spikes.md](spikes.md)).
