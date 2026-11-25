# Spikes

Status: accepted list for v1. A spike settles a risky premise before features depend on it. Each kill criterion was
written before the spike runs and MUST NOT be edited after results are known; a spike's PR adds its report to
`docs/spikes/<id>.md`. Spikes that need model calls quote their cost to the maintainer first (ADR 0011). Spikes that
need paid signing wait for the production-signing epic (ADR 0010).

Order of work: the cheapest test that could overturn a top-level decision comes first (§1–§13).

## 1. Core viability and build

| Id | Question | Kill criterion | OS | Duration |
|---|---|---|---|---|
| S-CORE3 | Does the real Rust workspace (core, protocol, providers, mcp, vault, db, policy, sandbox, egress) with all dependencies keep a fast build loop? | `cargo check -p lyra-core` after a one-line edit > 10 s, or a clean CI release build > 15 min on a GitHub-hosted runner (any OS) → split crates / sccache; still over → re-evaluate a hybrid with the Rust parts as separate processes | all | 0.5 d |
| S-CORE4 | Does relaying through Electron main hold 3 parallel turns at 200 tok/s each plus a 10 MB tool output? | Added p95 latency > 16 ms per delta batch, or main CPU > 15% → shared-memory or batched relay | all | 0.5 d |
| S-CORE5 | Can ACG, CFG and `NoRemoteImages` be enabled on `lyra-core.exe` with rusqlite, rustls and tree-sitter? | Any crash or failed TLS/DB operation → drop ACG and document the gap | Windows x64 + arm64 | 0.25 d |
| S-DB | Does `rusqlite` `bundled-sqlcipher` build with FTS5 on all six targets, and which crypto provider works on MSVC and arm64? | `bundled-sqlcipher-vendored-openssl` fails on win-x64 or win-arm64 in CI → libtomcrypt provider; neither builds on any target, or FTS5 is missing → SQLite3MC via `sqlite3mc-src` with own glue | all | 0.5 d |
| S-D1 | Does Forge 8 + Vite 8 + the fuses plugin produce an app whose fuses read back as specified, whose ASAR integrity trips on a 1-byte edit, and which spawns `lyra-core` from the bundle? | Any fuse not applied after packaging, integrity not enforced, or main-process cold start > 500 ms slower with the V8 snapshot fuse (then leave that fuse off). Ad-hoc signing only | all | 1 d |
| S-D2 | Can E2E run on a test-flavour build (inspect fuse on) while the release flavour keeps it off, with CI proving the release fuses? | The test flavour cannot be produced from the same source without manual steps, or Playwright cannot attach → WebdriverIO service or test-only IPC | all | 0.5 d |
| S-D4 | Do `tsc` 7.0.2 `--noEmit`, Vite 8 + plugin-react, Vitest 5 and type-aware Oxlint pass on a 3-package workspace with project references? | Any required tool needs the compiler API with no TS 6 alias workaround | all | 2 h |

## 2. Windows sandbox and hardened core

| Id | Question | Kill criterion | OS | Duration |
|---|---|---|---|---|
| S-W1 | Does `mxc-sdk` ProcessContainer hold R1–R5 with Lyra's policy, and does the S2 command set run? | Kill MXC if any of: a probe reaches a file outside the roots; a probe reaches the internet, another loopback port, `docker_engine`, `openssh-ssh-agent` or an Everyone-ACL pipe; DNS resolves without the proxy; > 6 of 30 commands need non-narrow escalation; ETW telemetry still routes with the build flag; `probe()` ever picks a non-`base-container` tier | Win 11 25H2 (.9550+), 24H2 (.9278), 26H1 arm64 | 3 d |
| S-W2 | Can `lyra-egress` run in an unpackaged AppContainer profile and serve as `allowedProxyPeer` with `hostLoopback: deny`? | If not, ship the identity-less mode only if a sandboxed connect to any non-proxy loopback port fails (and only in dev builds; release refuses it) | Win 11 | 1 d |
| S-W3 | Does an outer job (`ActiveProcessLimit` 256, `JobMemoryLimit`) nest with MXC's job, and does kill take the whole tree? | Any surviving descendant 2 s after kill, or limits not enforced → limits become a documented gap | Win 11 | 0.5 d |
| S-VAULT-W3 | Do the `OWNER RIGHTS` DACL and mitigation policies stop a same-user `WriteProcessMemory` / `CreateRemoteThread` into core? | A plain same-user test injector succeeds → record the gap and push to the enclave or service tier | Win 11 | 0.5 d |

## 3. Linux sandbox

| Id | Question | Kill criterion | OS | Duration |
|---|---|---|---|---|
| S-L1 | Bundled bwrap + AppArmor profile + seccomp + Unix-socket bridge: do R1–R6 hold? | Fails on any distro where the .deb/.rpm profile is installed → rethink (Landlock-only mode, disclosed as M); any probe succeeds → stop | Ubuntu 22.04, 24.04, Debian 12, 13, Fedora latest | 3 d |
| S-L2 | Landlock ABI detection and cgroup delegation (`systemd-run --user`) | No cgroup delegation on ≥ 2 of 5 distros → process limits are M there; disclose | same | 1 d |

## 4. macOS sandbox

| Id | Question | Kill criterion | OS | Duration |
|---|---|---|---|---|
| S-M1 | Lyra's SBPL generator vs a sandbox-runtime baseline: R1–R5 probes and the 30 commands | Any probe succeeds, or > 6/30 non-narrow escalations, or per-command overhead > 50 ms | macOS 15, 26, 27 | 2 d |
| S-M2 | Can a `setsid` + double-fork daemon escape the kill sweep, and does it keep network after token revocation? | If it keeps network, the proxy design is wrong (stop) | macOS | 0.5 d |
| S2 | Does the deny-default T1 policy support real work: 30 developer commands from 3 repos (npm/pnpm install and test, pytest, cargo build, git fetch/commit, playwright install, one Go tool for TLS trust; `docker build` excluded by design)? | > 6 of 30 need an escalation that cannot be expressed as a narrow domain or path grant → revisit the toolchain path model, or make T2 the default for coding | each OS (run inside S-M1, S-L1, S-W1) | in those spikes |

## 5. Egress proxy

| Id | Question | Kill criterion | OS | Duration |
|---|---|---|---|---|
| S-E1 | `lyra-egress` under a monorepo `pnpm install`, `pip install`, `cargo fetch`, `git fetch` with method/path rules; is the name-constrained CA accepted by Node, Python, Git-OpenSSL and Go? | RSS > 150 MB or > 1.5× wall time vs direct; any client rejects the name-constrained CA; an off-list name issued under the CA validates → fix, or fall back to the sandbox-runtime proxy patched for the CA | all | 3 d |

## 6. Identity, vault and IPC

| Id | Question | Kill criterion | OS | Duration |
|---|---|---|---|---|
| S-CORE2 | Electron main spawns a notarized `LyraCore.app`; core does put/get/delete in `TEAMID.dev.thehenrylabs.lyra.vault`; an ad-hoc copy gets -34018; the computer-use helper accepts `dev.thehenrylabs.lyra.core` and rejects an Electron Helper. Windows: the MSIX core is accepted and a squatted pipe aborts launch. Linux: the pidfd exe check passes | Any one fails → vault in a dedicated helper bundle. **macOS and MSIX parts need production signing**; the Windows squat and Linux pidfd parts run unsigned | all | 2 d |
| S-VAULT-M1 | Data-protection keychain item from the core process in a notarized Developer ID build | No topology yields put/get/delete from core, or another team's binary can read, or any password prompt in normal use → vault helper app. **Needs production signing** | macOS | 1–2 d |
| S-VAULT-M2 | Is a Secure Enclave-wrapped VMK usable under Developer ID without extra entitlement prompts? | Needs a restricted entitlement or prompts per launch → drop the SE wrap, keep the keychain item. **Needs production signing** | macOS | 0.5 d |
| S-VAULT-W1 | Platform Crypto Provider non-exportable key create/decrypt from a standard-user process; latency | Create fails on > 1 of 4 test machines (including a VM and an arm64 device), or unwrap p95 > 200 ms → DPAPI (L1) + service | Windows x64 + arm64 | 1 d |
| S-VAULT-W2 | `KeyCredentialManager` from an unpackaged exe: does it work, and is the key app-scoped? | A second unpackaged exe can `OpenAsync` the same credential → drop the app-scoped claim; keep Hello as a plain in-core gate | Windows | 0.5 d |
| S-VAULT-W4 | Named pipe with DACL + label: can a low-IL, AppContainer or restricted-token client open it; is a squatter detected? | The restricted client connects → inherited handles only; drop the CLI pipe | Windows | 0.5 d |
| S-VAULT-W5 | Does MSIX identity for the computer-use helper give per-app consent without breaking sandboxing or UIA? | UIA into normal windows or the signed-update path breaks, or a Store account is needed → Lyra consent card only | Windows | 1 d |
| S-VAULT-W6 | VBS-enclave seal/unseal of the VMK from Rust with Hello (v1.1 tier) | Needs unapproved spend, cannot run on a standard laptop with default VBS, or build/debug cost > 5 days → keep the service tier | Windows | 3–5 d |
| S-VAULT-L1 | Secret Service via `secret-service` 6.0.0: availability detection, locked-collection prompt, with GNOME Keyring, KWallet and KeePassXC | False "available" on any of 3 providers, or a locked collection hangs > 10 s with no timeout | Ubuntu 24.04, Fedora, Arch + KDE, Wayland and X11 | 1 d |
| S-VAULT-L2 | `systemd-creds --user` TPM2 seal/unseal as a standard user, without root; refusal in containers | Unseal needs root on Ubuntu or Fedora defaults → `tss-esapi` path | Linux | 0.5 d |
| S-VAULT-L3 | `SO_PEERPIDFD` + `/proc/<pid>/exe` check on kernels 5.15, 6.1, 6.8; fallback below 6.5 | Unsupported → `SO_PEERCRED` + inherited socketpairs only (documented) | Linux | 0.5 d |
| S-VAULT-V2 | Envelope format: test vectors, AAD tamper rejection, per-record key derivation (Rust only) | Any tampered AAD or record decrypts, or vectors are not stable across targets → fix the format before E3 ships | all | 0.5 d |

## 7. Providers and models

Model calls use the Claude API or a RunPod `Qwen/Qwen3.8-27B` endpoint (ADR 0011); cost is quoted first.

| Id | Question | Kill criterion | OS | Duration |
|---|---|---|---|---|
| S-CORE1 | Do Lyra's own adapters pass 20 scripted tasks (single tool, parallel tools, tool error, streaming, long context, reasoning replay, cache hit, abort, redirect refusal, allow-listed body dump)? Required targets: Claude API and vLLM + Qwen3.8-27B; OpenAI, Gemini, Vercel AI Gateway and LiteLLM when credentials exist | > 3/20 failures from adapter defects on any target, or the four formats exceed 7k LOC → TS provider worker | macOS | 3 d |
| S-AF6 | Rust reference adapters (`genai`, Rig) vs Lyra's on the S-CORE1 cases | An adapter that drops reasoning/thinking blocks, cache controls or parallel tool calls on ≥ 2 of 6 endpoints is out; if both are out, Lyra's own adapters stand | macOS | 1 d |
| S1-vLLM | Does the Chat-compatible adapter complete the 20 tasks on vLLM 0.34.0 + Qwen3.8-27B? Gate: vLLM must return well-formed `tool_calls` for a trivial single-tool task within 5 attempts with the recipe flags, else switch to SGLang or drop the self-hosted target | > 3 of 20 fail **because of adapter defects** (after attributing failures to parser or model via raw logs) | client on all three | 1 h GPU + 0.5 d |
| S1b-parsers | Do vLLM `qwen3_coder` (Python and Rust frontends) and SGLang return the same `tool_calls` for the same raw output, streaming and not? | Divergence on > 2 of 20 fault-injection fixtures → Lyra adds its own tool-call repair/validation layer and treats a missing tool call as retryable | any | 1 h GPU |
| S1c-reasoning | Do `reasoning_content` round trips and `preserve_thinking` work through the adapter? | History reasoning not honoured, or `reasoning_effort: xhigh` clamped → add `compat.allowFields` plumbing for `chat_template_kwargs` | any | 0.25 h GPU |
| S6-tunnel | Does a pinned-loopback endpoint work through an SSH tunnel, including abort and reconnect mid-stream? | Abort leaves the server generating (`vllm:num_requests_running` does not return to 0 within 5 s) | all | 0.25 h GPU |
| S7-cu | Optional: is Qwen3.8-27B grounded well enough for computer use? | < 60% of 50 grounded clicks land within the target element | any | 1.5 h GPU |

## 8. Durability, evals, telemetry, retrieval

| Id | Question | Kill criterion | OS | Duration |
|---|---|---|---|---|
| S-AF1 | Kill -9 core at 7 lifecycle points × 3 action classes | Any duplicate execution of a gated action, any lost approval, or any write after lease expiry → redesign before the coding agent | Windows (Defender on), macOS, Linux | 1.5 d |
| S-AF3 | Zero egress by default; export works | Any packet to a host outside the catalogue on a fresh install, or any `gen_ai.*.messages` attribute present while capture is off → block release | all | 0.5 d |
| S-AF4 | Eval harness through headless `lyra-core` (30 tasks + an AgentDojo subset) | Cannot run on all 3 OS runners in ≤ 20 min, or temperature-0 reruns disagree on > 10% of cases → own Vitest eval runner | all (Docker tasks on Linux) | 1 d |
| S-AF5 | Is vector retrieval worth it? Base, base + FTS5, base + FTS5 + sqlite-vec on 30 real questions | Hybrid must beat FTS5-only by ≥ 5/30 correct answers (blind-judged), with p95 query < 150 ms at 50k chunks; otherwise no vectors | macOS, then Windows arm64 build check | 1 d |

## 9. Browser

| Id | Question | Kill criterion | OS | Duration |
|---|---|---|---|---|
| S3 | Can users sign in and finish one read-only task on GitHub, a Google account, Atlassian Cloud and one 2FA site in the Electron partition (option A), and does option B keep sessions across the relaunch? | Google or ≥ 2 others block sign-in in A → option B (login launch without debugging, then pipe relaunch) becomes the default, A stays for non-auth browsing. Automation masking is not used to pass | all | 2 d |
| S3a | Does a `webContents.debugger` transport satisfy Playwright `connectOverCDP`? | Cannot complete `newPage`, `click` and `ariaSnapshot` on one page in 1 day → raw typed CDP for A | all | 1 d |
| S3b | Is a Lyra Chrome profile valid after relaunch with the pipe; does Edge honour the default-dir rule; is `RemoteDebuggingAllowed=false` detected? | Relaunch invalidates the session on any OS, or Edge ignores the rule (then forbid Edge's default profile) | Win, macOS, Linux | 1 d |
| S3c | Electron and Chrome .deb with the shipped AppArmor profile: is the sandbox active? | Sandbox cannot be enabled without `--no-sandbox` or a global sysctl → Linux ships with the browser tool disabled until fixed | Ubuntu 24.04, 26.04 | 1 d |
| S3d | Electron WebAuthn platform authenticator and `safeStorage` backend per OS | Passkey fails in A on Windows or Linux → document "use option B for passkeys" | all | 0.5 d |

## 10. Shell, packaging and update

| Id | Question | Kill criterion | OS | Duration |
|---|---|---|---|---|
| S-SH1 | Does a Forge-built, Authenticode-signed MSIX install, start core, run the sandbox runner and apply an update from the Lyra manifest? | Fails any of: install on clean Win 11 x64 and arm64 without manual cert steps; core starts; update applies and a lower `sequence` is rejected → per-machine MSI. **Needs production signing** (a test-certificate pre-check MAY run earlier) | Win 11 x64 + arm64 | 2 d |
| S-SH2 | UI parity: CSP matrix, xterm WebGL fallback, fonts, screen readers (NVDA, Orca), Wayland kill-switch hotkey and modal approval dialog | Any CSP violation not seen on macOS; a screen reader cannot reach the approval dialog; no reliable stop control on GNOME Wayland (then an in-app stop plus a documented limitation) | Windows, Ubuntu 24.04 Wayland + X11, Fedora KDE Wayland | 3 d |
| S-SH4 | Idle RAM and cold start of shell + core + one agent tab vs a Tauri 2 hello-world | Electron idle total > 2× the Tauri baseline **and** > 600 MB → reconsider process counts (the shell choice stands) | all | 1 d |
| S-SH5 | Linux update chain: signed apt and rpm repos end to end; manifest-verified notifier; AppArmor profile for bwrap | The package manager refuses the key, or bwrap is still blocked after profile install | Ubuntu 24.04, Fedora | 1 d |
| S-SH6 | Watch item: Tauri 3 CEF with a Windows sandbox and no `disable-library-validation` | Re-run only when `tauri-runtime-cef` reaches 3.0.0 stable | Windows, macOS | 1.5 d |
| S-D3 | Does Squirrel.Mac refuse an update signed by another identity, and does the manifest check catch a version mismatch and a downgrade? | A downgrade or substituted bundle reaches `quitAndInstall` → evaluate alternatives. **Needs production signing** | macOS | 1 d |
| S-SC5 | Manifest verifier: replay, downgrade, expired and truncated manifests | Any accepted | all | 0.5 d |

## 11. Computer use

| Id | Question | Kill criterion | OS | Duration |
|---|---|---|---|---|
| S-X | Cross-OS conformance harness: 40 fixture cases with focus, z-order and leak oracles | Harness cannot be written in 4 d → ship macOS + Windows first, Linux behind a flag | all | 4 d |
| S5 | Swift helper: screenshot, AX tree, click, type, tier check by target PID, TCC attribution via `SMAppService`; 10 tasks in Finder, Notes and a terminal | Any wrong-target event in 200 scripted actions → AX-element actions only, no raw coordinate clicks | macOS | 3 d |
| S-Win-1 | WGC per-HWND capture of an allowed window while another covers it; border appears; no other pixels leak | Any pixel of a non-allowed window in the frame, or a covered window captures black in > 2 of 10 apps | Windows | 1.5 d |
| S-Win-2 | UIA actions on Notepad, File Explorer, a WPF app and an Electron app; elevated app refused; `SendInput` PID pre-check | Any wrong-target event in 200 scripted actions → UIA only, no pixel input | Windows | 2 d |
| S-Win-3 | Do low-level hooks tell Lyra-injected from user input; Esc kill ≤ 100 ms? | Cannot tell them apart in 50 trials → pause on any input, including Lyra's, via the lease | Windows | 1 d |
| S-Lin-1 | X11 nested Xephyr: app inside, capture only that server, input only via XTest there | The app needs host resources (clipboard, GPU, portals) that break in Xephyr for 3 of 5 apps → VM desktop mode only on X11 | Linux X11 | 1.5 d |
| S-Lin-2 | GNOME 49/50/51 Wayland: ScreenCast window grant with restore token, AT-SPI actions, libei foreground lease on GTK4 and Electron | The restore token does not persist, or no wrong-focus guard is possible → element actions only on GNOME | Ubuntu 24.04+ GNOME | 2.5 d |
| S-Lin-3 | KDE Plasma 6: ScreenCast window grant, AT-SPI, RemoteDesktop dialog | Portal dialog missing or capture monitor-only → capture and AT-SPI only (Tier 3) | KDE | 1.5 d |
| S-Lin-4 | `cua-driver-rs` `platform-linux` at a tag as a fallback, wrapped default-deny | Cannot be wrapped default-deny without patching > 2 kLOC → drop adoption | Linux | 2 d |

## 12. UI, MCP, engines, supply chain

| Id | Question | Kill criterion | OS | Duration |
|---|---|---|---|---|
| S-UI-1 | Can locked-down Streamdown + Shiki render a recorded 25k-character code-heavy stream at 4× CPU throttle without long frames? | p95 frame > 33 ms, any frame > 100 ms, or gzip markdown chunk > 300 KiB → fallback pipeline (marked lexer + memoized react-markdown blocks + own Shiki component), re-measured with the same criterion | macOS, then all | 1 d |
| S-UI-2 | With the target CSP (including Trusted Types), do editor, diff, terminal, Base UI, KaTeX and Shiki produce zero violations? | Any violation needing `style-src-elem 'unsafe-inline'` in the main window (the terminal's own frame may allow it) | all | 0.5 d |
| S-UI-3 | XSS corpus through `<LyraMarkdown>`, terminal and diff panes with a CSP reporter | Pass only with zero script execution, zero network requests and zero navigations | all | 0.5 d |
| S-MCP1 | Does all MCP traffic, including OAuth discovery, token and CIMD/DCR calls, go through core's pinned HTTP client, and does version negotiation work against 5 real servers (2025-06-18 stdio, 2025-11-25 HTTP, 2026-07-28, Atlassian, GitHub)? | Any request escapes the client, or negotiation fails on > 1 of 5 → patch or fork the `rmcp` transport/auth module | macOS | 0.5 d |
| S4 | Can Lyra drive external engines over ACP without touching their credentials? | An engine that runs a write/exec tool without `session/request_permission` is labelled "approval not enforceable" or dropped; no documented interface or a terms conflict → drop that engine | macOS | 1 d |
| S-SC1 | Can the full app be signed and notarized with the key on a hardware token? | Unresolved errors after 2 h, > 5 touches or > 10 min per signing → non-extractable key in a separate keychain; update key stays on hardware. **Production-signing epic** | macOS | 0.5 d |
| S-SC2 | Do two clean builds produce the same `app.asar` and core hashes? | Differences not removable within 0.5 d after fixing timestamps and file order → "signed SBOM + hashes only" | all | 0.5 d |
| S-SC3 | Does osv-scanner catch a known-CVE package and a known-malicious package in a throwaway lockfile? | It misses either → add a second scanner as a required gate | Linux | 2 h |
| S-SC4 | gitleaks vs betterleaks on 30 seeded fake secrets | Switch only if betterleaks finds ≥ 3 more with no more false positives | Linux | 1 h |

## 13. VM tier

| Id | Question | Kill criterion | OS | Duration |
|---|---|---|---|---|
| S-VM | microsandbox: boot, worktree copy, egress only via `lyra-egress`, no TSI | Cold boot > 5 s; any direct egress; virtiofs `npm install` > 3× host; Windows build not working → Windows T2 gap, libkrun or `containerization` elsewhere | macOS (Apple silicon), Linux (KVM), Win 11 (WHP) | 3 d |

## 14. Id mapping

Earlier drafts prefixed spike ids with the number of their source document. The plain ids above replace them.

| Former | Id | Former | Id |
|---|---|---|---|
| 10-S-AF1, -AF3..AF6 | S-AF1, S-AF3..S-AF6 | 14-M1, 14-M2 | S-VAULT-M1, S-VAULT-M2 |
| 10-S-AF2 | dropped (LangGraph bake-off; no TS core) | 14-W1..W6 | S-VAULT-W1..S-VAULT-W6 |
| 11-S-CORE1..5 | S-CORE1..S-CORE5 | 14-L1..L3 | S-VAULT-L1..S-VAULT-L3 |
| 11-S-DB, 14-D1 | S-DB | 14-L4 | dropped (Flatpak not in v1) |
| 12-W1, W2, W4, W5, W6 | S-SH1, S-SH2, S-SH4, S-SH5, S-SH6 | 14-D2 | dropped (cross-engine DB open; Rust core only) |
| 12-W3 | dropped (N-API prebuild matrix; covered by S-CORE3 and S-DB) | 14-V2 | S-VAULT-V2 (Rust-only scope) |
| 13-S-W1..W3, S-M1, S-M2, S-L1, S-L2, S-E1, S-VM | same without prefix | 15-S3a..S3d, 15-S5 | S3a..S3d, S5 |
| 15-S-Win-1..3, S-Lin-1..4, S-X | same without prefix | 16-S1-vLLM, S1b, S1c, S6, S7 | S1-vLLM, S1b-parsers, S1c-reasoning, S6-tunnel, S7-cu |
| 01-S-D1..D4 | S-D1..S-D4 | 02-S-UI-1..3 | S-UI-1..S-UI-3 |
| 07-S1..S5 | S-SC1..S-SC5 | 03-S2 | S-MCP1 |
| design S1 | S-CORE1 | design S2, S3, S4, S5 | S2, S3, S4, S5 |
| design S2b | S-VM | 03-S3 (AI SDK loop parity) | dropped (no AI SDK loop) |
