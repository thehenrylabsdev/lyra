# Technology stack

Status: accepted for v1. Versions were current on 2026-11-25. Before adding or bumping a dependency, re-check the
newest stable release and the 7-day release-age rule (§5). **Build** marks a component Lyra writes itself.

## 1. Platforms and OS floors

| OS | Floor | Packaging | Notes |
|---|---|---|---|
| macOS | **15+** (arm64; x64 optional) | DMG + ZIP (Squirrel.Mac) | Covers Electron 45, ScreenCaptureKit and `SMAppService`. The VM fallback (`containerization`) needs macOS 26 |
| Windows | **Windows 11 at the MXC build floor**: 24H2 26100.9278+, 25H2 26200.9278+, 26H2 26300.9550+, 26H1 28000.2804+ (new arm64 devices). **.9550+ recommended** (loopback servers inside the container). x64 and arm64 | **MSIX** (+ MSI for enterprise per-machine installs) | **No Windows 10.** Older builds get `sandbox_unsupported`; the coding agent is disabled, not weakened |
| Linux | Ubuntu 22.04 / 24.04+, Debian 12 / 13, current Fedora; x64 and arm64 | **.deb / .rpm only**, from signed apt/dnf repositories, installed to root-owned `/opt/Lyra` | Kernels before 6.5 lack `SO_PEERPIDFD`: fall back to `SO_PEERCRED` and inherited socketpairs. Vault L2 needs systemd ≥ 256 (Debian 13, current Fedora) |

**Not offered in v1:** Flatpak (its sandbox blocks the nested user namespaces bubblewrap needs), AppImage (cannot
install AppArmor profiles or a root-owned install), Snap, Windows 10.

## 2. Picks

| Layer | Pick (exact) | Licence |
|---|---|---|
| Shell | Electron **45.5.0** (Node 24.22.3, Chromium 156), Electron Forge 8.0.1 (makers: dmg, zip, msix, wix, deb, rpm), `@electron/fuses` 2.1.3, `@electron/asar` 4.3.4, `@electron/windows-sign` 2.1.0, `@electron/notarize` 3.1.2, `@electron/osx-sign` 2.7.2 | MIT |
| UI | React 19.3.1, Tailwind 4.3.5, Base UI 1.9.0 (`CSPProvider` nonce), Zustand 5.1.0, TanStack Query 5.108.4, Streamdown 2.8.1 (locked down: no raw HTML, protocol allowlist, no remote images) with `@streamdown/code` 2.0.1 and `@streamdown/math` 1.0.4, Shiki 4.6.0 (JS regex engine), KaTeX 0.16.47, virtua 0.56.0, CodeMirror 6 (`@codemirror/view` 6.44.0, `state` 6.7.6, `merge` 6.13.0), xterm.js 6.0.0 (+ addon-fit 0.11.0, addon-webgl 0.19.0, own CSP frame), lucide-react 1.80.0, react-resizable-panels 4.24.2 | MIT / Apache-2.0 |
| TS toolchain (UI and main only) | pnpm 12.23.0, Vite 8.3.8 + `@vitejs/plugin-react` 6.3.0, TypeScript 7.0.2, Oxlint 1.95.0, Oxfmt 0.80.0, Vitest 5.1.2, Playwright 1.65.1 | MIT / Apache-2.0 |
| Rust toolchain | Rust **1.100.0** pinned in `rust-toolchain.toml` | MIT / Apache-2.0 |
| Core runtime | `tokio` 1.54.1, `serde_json` 1.0.154, `reqwest` 0.13.5 (rustls, stream, http2) | MIT |
| Protocol | **Build**: own JSON-RPC (~0.5k LOC); `schemars` 1.2.3 → JSON Schema → `ts-rs` 12.0.1 (`serde-json-impl`); CI fails on drift | MIT |
| Providers | **Build**: four wire formats (~4–7k LOC). `genai` 0.7.5 as a test oracle only; `tiktoken-rs` 0.12.1 for estimates | MIT |
| Agent loop + durability | **Build**: own loop (~2–3k) + durable turn state machine over the event log (~1–1.5k) | — |
| MCP | `rmcp` **3.11.0** (client features only), `jsonschema` 0.63.1 | Apache-2.0 / MIT |
| ACP | `agent-client-protocol` 5.6.1, no `unstable_*` features | Apache-2.0 |
| DB | `rusqlite` 0.43.0 + `bundled-sqlcipher` (`libsqlite3-sys` 0.40.0, `sqlcipher-src` 420.0.0 = SQLCipher 4.20.0 / SQLite 3.54.1), FTS5; crypto provider per OS decided in spike S-DB | MIT / BSD-3 |
| Vault | **Build** `lyra-vault` (~0.9k) + `lyra-harden` (~0.25k) on `keyring-core` 1.1.0, `apple-native-keyring-store` 1.0.3 (`protected`), `windows` 0.62.2 (CNG Platform Crypto Provider, WinTrust, pipes), `secret-service` 6.0.0, `systemd-creds` (TPM2); XChaCha20-Poly1305 (RustCrypto `chacha20poly1305`), `zeroize` | MIT / Apache-2.0 |
| Local IPC auth | **Build** `lyra-peer` (~0.6k): macOS audit token (`security-framework` 3.7.0 + own FFI), Windows pipe DACL + `FIRST_PIPE_INSTANCE` + Authenticode, Linux `SO_PEERPIDFD` (`rustix` 1.1.6 / `nix` 0.32.0) | — |
| Policy + command parser | **Build** (~2.5–3.5k) on native `tree-sitter` 0.27.2 + `tree-sitter-bash` 0.25.1; `proptest` | MIT |
| Sandbox | **Build** `lyra-sandbox` (~12–13k): `landlock` 0.4.8, `seccompiler` 0.5.0, `nix` 0.32.0, `windows-sys` 0.61.2, **`mxc-sdk` =1.1.0** (default features, built with telemetry unrouted), bundled **bubblewrap 0.13.1**. SBPL and seccomp designs copied from `openai/codex` with attribution | MIT / Apache-2.0; bubblewrap LGPL-2.0+ (separate binary, source offer) |
| Egress | **Build** `lyra-egress` (~4–5k): `hyper` 1.13.0, `rustls` 0.23.46, `rcgen` 0.14.10 (`NameConstraints`), `hickory-resolver` 0.26.4 | MIT / Apache-2.0 |
| VM tier | libkrun 1.19.8 via microsandbox 0.8.2 (after spike S-VM); fallback Apple `containerization` 0.57.0 (macOS 26) | Apache-2.0 |
| Browser | `playwright-core` 1.65.1 in Electron main (partition over `webContents.debugger`, or Chrome/Edge over `--remote-debugging-pipe`); `devtools-protocol` types (dev) | Apache-2.0 / BSD-3 |
| Computer use | **Build**: `lyra-cu-proto` (~1.5–2k Rust); macOS Swift + AXorcist 0.2.3 (~3k); Windows Rust `windows` 0.62.2 (~5k); Linux Rust `ashpd` 0.14.3, `pipewire` 0.10.1, `reis` 0.7.1, `atspi` 0.30.0, `x11rb` 0.14.0 (~7–10k). `accesskit` 0.25.2 for Lyra's own UI accessibility only | MIT / Apache-2.0 |
| Telemetry | `tracing` 0.1.44; `tracing-opentelemetry` 0.34.0 + `opentelemetry-otlp` 0.33.1 behind an off-by-default feature; OTel GenAI semantic conventions v0.1.0 | MIT / Apache-2.0 |
| Retrieval | SQLite FTS5 now; `sqlite-vec` 0.1.9 statically linked only if spike S-AF5 passes | public domain / MIT-Apache |
| Evals (CI/dev only) | promptfoo 0.126.0, Inspect AI 0.3.292 (CI); garak 0.17.1, PyRIT 1.2.1 (dev); separate `evals/` lockfile | MIT / Apache-2.0 |
| Supply chain | Rust: `cargo-deny` 0.22.1, `cargo-audit` 0.23.0, `cargo-vet` 0.10.3, `cargo-auditable` 0.7.8, `cargo-cyclonedx` 0.5.10, `cargo-nextest` 0.9.152, `cargo vendor` + offline CI, crates.io-only sources, no git dependencies. JS: pnpm `minimumReleaseAge` + `trustPolicy: no-downgrade` + `allowBuilds` allowlist, osv-scanner 2.7.1. Also Syft 1.57.2 (SBOM), zizmor 1.33.1 (workflows), gitleaks 8.30.1, actions pinned by SHA | MIT / Apache-2.0 |
| Model test bed | `Qwen/Qwen3.8-27B` (Apache-2.0, dense 27.8B, vision-language) on vLLM 0.34.0 or SGLang 0.5.23; or the Claude API (ADR 0011) | Apache-2.0 |
| Signing and update | **Later epic.** macOS Developer ID + notarization; Windows Authenticode (Azure Artifact Signing or OV on a cloud HSM); Linux GPG for repos. Lyra's own Ed25519 manifest (version, sequence, expiry, hashes) drives Squirrel.Mac and MSIX; on Linux apt/dnf install and the app only notifies. Dev builds are unsigned or ad-hoc (ADR 0010) | — |

## 3. Per-OS security parity (summary)

Strength: **S** OS- or kernel-enforced · **M** enforced with known caveats · **W** weak. Details in
[sandbox.md](sandbox.md), [secrets.md](secrets.md) and [browser-and-computer-use.md](browser-and-computer-use.md).

| Control | macOS | Windows | Linux |
|---|---|---|---|
| Core identity (signed release) | `LyraCore.app` with own profile and keychain group **S** | Authenticode + MSIX package identity **S/M** (MSI: no package identity, **M**) | Root-owned `/opt/Lyra` + signed hash list **W** |
| Shell → core channel | Inherited stdio **S** | libuv stdio pipes with default DACL **M** | Inherited stdio **S** |
| Vault (VMK custody) | Data-protection keychain **L3** (dev build: L1) | TPM-bound CNG key **L2**; DPAPI **L1** with opt-in | `systemd-creds` + TPM2 **L2** (systemd ≥ 256); Secret Service **L1** |
| Sandbox read/write deny | Seatbelt **S** | MXC BaseContainer **M** | bwrap + Landlock **S** |
| Network only via proxy | **S** | WFP + `allowedProxyPeer` **S** (identity-less mode refused in release) | `--unshare-net` + bridge **S** |
| Sockets / pipes | **S** | pipe DACLs **M (unverified)** | **S** |
| Process limits / tree kill | **W** | nested jobs **S** | PID namespace + cgroup v2 **S/M** |
| Renderer sandbox, CSP, fuses | **S** | **S** | **S** (needs the .deb/.rpm AppArmor profile) |
| App integrity | ASAR integrity + notarization **S** | ASAR integrity + MSIX **S** | no ASAR integrity **M** → root-owned install + hash list |
| Capture limited to allowed apps | **S** | per window **M** | portal per window **M**; X11 **W** (Xephyr only) |
| OS consent for computer use | TCC on helper **S** | none **W** → Lyra consent card | Wayland portal **M**; X11 none **W** |
| Kill switch | event tap **S** | low-level hooks **M** | Wayland **W** → overlay Stop + portal shortcut |
| Agent browser | **S** | **S** (Edge fallback) | **S**; `basic_text` → ephemeral partition |

## 4. What Lyra builds

| Component | LOC (estimate, excluding tests) |
|---|---|
| Core: loop, durability, protocol, providers, MCP/ACP wrappers, DB, policy, telemetry | 15–23k |
| Sandbox + egress | ~17k |
| Vault, peer auth, hardening | ~2k |
| Computer-use helpers (three OSes) + conformance harness | 17–26k |
| Electron main (relay, launcher, dialogs, browser executor) | ~1–2k TS (UI not counted) |
| **Total** | **~52–70k** Rust/Swift |

Phasing carries this size: v1 ships core, sandbox, vault and browser on three OSes; computer use ships macOS and
Windows first, Linux behind a flag until spike S-X passes.

## 5. Dependency rules

1. **Release age.** Pins must be at least 7 days old at install time. Exception: `electron` and `@electron/*`
   security releases, reviewed by hand against release notes. Fast-moving packages are pinned to the newest version
   at least 7 days old at scaffold time. Cargo has no native release-age gate, so CI rejects new lockfile entries
   younger than 7 days.
2. **Rust build-time code** (`build.rs`, proc macros) is covered by `cargo-vet` audits, vendored offline builds and no
   git dependencies.
3. `rmcp` client features only; all HTTP through Lyra's `reqwest`; `cargo tree -d` checked in CI.
4. The `windows` umbrella crate is pinned at 0.62.2 while upstream moves to 0.100; prefer `windows-sys` and plan one migration.
5. Reproducible builds: `--remap-path-prefix` and `-ffile-prefix-map` in release; CI builds twice and compares hashes.
6. Size budget: CI fails if a stripped core binary exceeds 40 MB per target.
7. No copyleft runtime dependency except bubblewrap (separate binary, LGPL source offer). Avoided as not open source or
   incompatible: Arize Phoenix (ELv2), Mastra `ee/`, Sentry (FSL), Screenpipe, AGPL perception modules.
8. Every Electron security release ships within 7 days; track the newest stable Electron line.

## 6. Top risks

| Id | Risk | Mitigation |
|---|---|---|
| RK-1 | Build volume (~52–70k LOC) | Phasing; copy proven open-source designs with attribution; golden-file tests for generated profiles; one cross-OS probe suite |
| RK-2 | MXC is new (GA 2026-10), its OS API is not on Microsoft Learn, it has required telemetry, and its floor excludes Windows 10 | Exact `probe()` gate; ETW check in CI; pinned SDK; read the diff on each bump; privacy-notice disclosure; fallback own AppContainer + LPAC backend |
| RK-3 | Provider adapters are Lyra's to maintain | Four formats only; recorded fixtures; `genai` oracle; fallback TS provider worker with no vault access |
| RK-4 | Per-user secret stores on Windows and Linux | Assurance levels in the UI; hardened core; VBS-enclave tier in v1.1 |
| RK-5 | `rmcp` advisory cadence; unmaintained transitive `chrono` | Client features only; `cargo-deny` gate with a time-boxed exception |
| RK-6 | Wayland computer use is fragmented; X11 has no isolation | Tiers in the UI; Xephyr or VM mode; Linux CU behind a flag |
| RK-7 | `sandbox-exec` deprecated on macOS | `sandbox_init` FFI fallback; VM tier |
| RK-8 | Electron lags Chrome stable by about three majors | Track the newest Electron line; 7-day security-release policy |
