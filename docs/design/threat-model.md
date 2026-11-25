# Threat model

Status: accepted for v1. Threat ids `L-01`..`L-42` are stable and are referenced from the roadmap, tickets and tests.

## 1. Risk profile

Lyra concentrates every credential a user has (model keys, GitHub, Google, Slack, Atlassian, browser sessions) and
drives a shell, a browser and the desktop. That aggregation makes it a higher-value target than any single-purpose
app. Design consequence: **assume prompt injection will sometimes succeed**, and contain it with OS boundaries,
code-enforced gates and no ambient secrets.

**Assets:** vault secrets; agent browser sessions; workspace source; conversation history; the user's desktop;
Lyra's update and signing keys.

**Adversaries:**

| Id | Adversary |
|---|---|
| ADV-1 | Injected content in web pages, files, MCP results and chat messages |
| ADV-2 | A confused or jailbroken model |
| ADV-3 | A malicious repository (config, hooks, `.env`, tasks) |
| ADV-4 | Malicious or compromised MCP servers, plugins and connectors (supply chain) |
| ADV-5 | Other local software running as the same user |
| ADV-6 | Network attackers (DNS rebinding, MITM, compromised gateways) |
| ADV-7 | Compromise of Lyra's build, update or signing pipeline |

## 2. Threat register

Test ids: `ST-*` and `A*` in [secrets.md](secrets.md) §9, `SB-*` in [sandbox.md](sandbox.md) §8, `BC-*` in
[browser-and-computer-use.md](browser-and-computer-use.md), `MT-*` in [mcp-and-connectors.md](mcp-and-connectors.md) §7,
`PR-*` in [providers.md](providers.md) §8, `H*` in [architecture.md](architecture.md) §10.

| Id | Sev | Threat to Lyra | Control | Test |
|---|---|---|---|---|
| L-01 | High | Secrets stored in plaintext, or silently falling back to plaintext when the OS store is missing | Envelope vault, no fallback, L0 refuses ([secrets.md](secrets.md) §1) | ST-1, ST-2, A1 |
| L-02 | High | The sandbox confines writes but leaves reads and network open, so injected code exfiltrates | Deny-default read, write and network; mandatory read-deny (R1–R3) | SB-1, ST-3 |
| L-03 | High | MCP servers, hooks, plugins or file tools run outside the sandbox | Everything the agent executes runs under T1 (R8) | MT-3 |
| L-04 | High | Repo-level config redirects the provider base URL, proxy, headers or env | Endpoint settings are user-layer only ([architecture.md](architecture.md) §8) | ST-5, PR-4 |
| L-05 | High | Command approval by regex or prefix is bypassed with chaining, substitution or wrappers | tree-sitter parse; every simple command must match; wrappers opaque | SB-3 |
| L-06 | High | An LLM approver is steered into approving a dangerous action | Judge is advisory; class floors for `transmit`/`destructive`/`financial`/`permission` | Injection corpus: judge-approved gated actions = 0 |
| L-07 | High | Browser or computer-use safety lives only in prompt text | Code-level origin allow-list and native confirmations | BC-1 |
| L-08 | High | The model gets raw CDP, JS evaluation or cookie access | Narrow browser API; no cookie, storage or JS tools | BC-2 |
| L-09 | High | Broad default access to personal data (screen, input, microphone, messages) | Per-capability opt-in; empty allowed-apps default | BC-4 |
| L-10 | Med-High | Weak Electron fuses allow code injection (RunAsNode, `NODE_OPTIONS`, inspect, no ASAR integrity) | Full fuse set, CI check on every artifact | H1 |
| L-11 | Med | `file://` extra privileges or `<webview>` in the shell | Fuse off; `WebContentsView` only | H1, H3 |
| L-12 | Med | Conversation history and search index stored unencrypted or world-readable | SQLCipher DB, owner-only permissions, redaction on write | A10, A14 |
| L-13 | Med | Local servers with shared TLS certs, open ports or tokens on argv let other software drive Lyra | Inherited channels; peer auth + pairing nonce; no TCP control ports | ST-6, A7, A8 |
| L-14 | Med | Fixed-port OAuth listeners or custom-scheme callbacks are intercepted | PKCE + `state` bound to the window; ephemeral ports | MT-6 |
| L-15 | Med | Remote code is hot-loaded outside the signed bundle, or an update is rolled back | No hot-loaded code; signed manifest with monotonic sequence | H13 |
| L-16 | Med | Unsigned in-process plugins or connector packages run with Lyra's privileges | Out-of-process plugins; signatures and hash pins | Tampered plugin refused |
| L-17 | Med | SSRF from agent fetches to loopback, LAN or cloud metadata | Proxy blocks private ranges after DNS, pins the IP | Fetch `169.254.169.254` and `127.0.0.1:*` fails (SB-1) |
| L-18 | Med | BYOK keys relayed through a backend instead of going to the user's endpoint | Direct client-to-endpoint calls | Capture shows keys only to the configured host |
| L-19 | Med | Vendor-hosted MCP placement holds user tokens | Local placement only | n/a (feature absent) |
| L-20 | Med | A model-controllable parameter disables the sandbox | No such parameter; T0 is human-only | SB-9 |
| L-21 | Med | Deep links trigger installs or prompts silently | Validated, confirmed deep links | H12 |
| L-22 | Med | Computer-use tier is checked on the frontmost app, then the event lands elsewhere (TOCTOU) | Check the target PID at event time | BC-5 |
| L-23 | Med | Arbitrary script primitives in computer use | Not offered; mediated batches only | BC-7 |
| L-24 | Med | The agent edits Lyra's policy or the repo's hook and IDE config | Mandatory write-deny | SB-6 |
| L-25 | Med | Browser native-messaging manifests become user-writable persistence | Write-deny `NativeMessagingHosts`; no native host in v1 | Sandbox write test |
| L-26 | Med | Telemetry and device ids on by default | Off by default; egress catalogue | S-AF3 zero-egress check |
| L-27 | Med | Lyra itself calls vendor hosts even behind a gateway | No Lyra-originated vendor calls | Capture with only a LAN vLLM: no other hosts |
| L-28 | Low-Med | Plain-HTTP update fallback or arbitrary-load exceptions | HTTPS only; ATS default | H10, H13 |
| L-29 | Low | DevTools, internal hosts, source maps or test origins left in release builds | Release stripping and audit | H9 |
| L-30 | Low | Secrets pasted by users persist in history or paste caches | Redaction on write; paste cache off | Seeded-secret test (A15) |
| L-31 | Med | A third-party gateway the user configures is itself vulnerable | Not bundled; treated as untrusted; hardening rules if ever bundled | Version gate at build |
| L-32 | Med | Messaging connectors become injection and exfiltration channels | `transmit` floor, no bulk sends, no auto-reply | MT-4 |
| L-33 | Med | Masking automation signals to defeat site bot detection breaks site terms and trust | Off by default; explicit per-site choice | Config audit |
| L-34 | Med | An MCP tool's description or schema changes after consent ("rug pull") | Hash pins; re-consent on change | MT-2 |
| L-35 | High | Approvals flow through the untrusted renderer, so one XSS approves everything | Main-process native dialogs + single-use HMAC token | Forged `approval/respond` without a token is rejected |
| L-36 | High | Project config widens policy (allow rules, domains, roots) | Project layer can only narrow | Repo with `allow: Browser.transmit(*)` has no effect |
| L-37 | High | The egress proxy becomes an exfiltration route or confused deputy (push to an attacker repo; lent credentials) | Fetch-only method/path rules; no injection in v1; push/publish are core-performed `transmit` | SB-4 |
| L-38 | High | Docker or other local sockets give a sandbox escape | Unix sockets and pipes denied (R5) | SB-5 |
| L-39 | Med | The VM shares the live workspace, so planted hooks or tasks later run on the host | T2 works on a copy; sync back through the deny filter | SB-7 |
| L-40 | Med | Computer-use grants attach to the main app; full-display screenshots leak other windows | Separate helper identity; per-app capture filters | BC-6 |
| L-41 | Med | After login, the agent reaches account-security or OAuth-consent pages | `permission` class for those pages | BC-3 |
| L-42 | Med | Signing or update keys are compromised (ADV-7) | Signing and update keys on hardware tokens, never on CI disk; the update manifest key is separate and offline; notarization required for macOS releases | Release checklist |

### 2.1 Public references

| Id | Reference |
|---|---|
| L-10 | Electron fuses: https://www.electronjs.org/docs/latest/tutorial/fuses |
| L-11 | Electron security checklist: https://www.electronjs.org/docs/latest/tutorial/security |
| L-17 | Instance metadata endpoint `169.254.169.254` (cloud providers' public docs) |
| L-25 | Chrome native messaging host locations: https://developer.chrome.com/docs/extensions/develop/concepts/native-messaging |
| L-31 | LiteLLM advisories CVE-2026-42208 (SQL injection) and CVE-2026-42271 (MCP command execution): https://github.com/BerriAI/litellm/security/advisories |
| L-33 | Chrome remote-debugging changes: https://developer.chrome.com/blog/remote-debugging-port |
| L-38 | Docker daemon socket access is root-equivalent: https://docs.docker.com/engine/security/ |

## 3. Per-OS residual gaps

| Gap | OS | Compensation |
|---|---|---|
| Secrets are per-user, not per-app (L2/L1) | Windows, Linux | Hardened core, presence gate, sandbox denies keystore, level shown in the UI ([secrets.md](secrets.md) §2) |
| Same-user processes indistinguishable by the OS | Linux | Pairing nonce over an inherited fd; approvals only via native dialog |
| Weak process limits and tree kill | macOS | Token revocation at the proxy; descendant sweep; T2 for hard limits |
| MXC hard-link aliases and Schannel trust | Windows | Refuse protected files with link count > 1; fail closed for Schannel tools on MITM'd hosts |
| No ASAR integrity | Linux | Root-owned `/opt/Lyra`; launcher checks a signed hash list before starting core |
| No OS consent for UI automation | Windows, Linux X11 | Lyra consent card before every capability; X11 apps only in nested Xephyr |
| Unsigned dev builds | all | Lower, labelled vault level on macOS (L1); peer auth by nonce only; TCC grants reset per build; updater off ([architecture.md](architecture.md), ADR 0010) |

## 4. Accepted residual risks (revisit)

- **Prompt injection is not solved.** Controls reduce the blast radius; they do not stop the model being steered inside
  allowed actions (e.g. writing wrong code in the workspace). Mitigation: checkpoints, diff review, CI.
- `sandbox-exec` is deprecated on macOS; a future release could remove it. Hedge: `sandbox_init` FFI, then T2.
- Same-user malware (ADV-5) can read Lyra's encrypted DB files and, on Windows and Linux, ask the keystore to unwrap the VMK.
- Agent browser profile cookies are as exposed to ADV-5 as any browser's.
- External engines carry their own weaknesses; Lyra wraps them but cannot fix them, and labels their runs.

## 5. Security process

- Every PR touching core, helpers, IPC, vault, sandbox, egress or connectors gets an independent security review and
  the `security-critical` label.
- Release gate: fuse check, release audit (H9), dependency advisory check (`cargo-deny`, `cargo-audit`, osv-scanner),
  and the acceptance tests in [secrets.md](secrets.md) §9, [sandbox.md](sandbox.md) §8 and
  [mcp-and-connectors.md](mcp-and-connectors.md) §7 on all three OSes.
- Secret scanning (gitleaks) on every commit and PR.
- Vulnerabilities are reported privately through GitHub's private vulnerability reporting.
