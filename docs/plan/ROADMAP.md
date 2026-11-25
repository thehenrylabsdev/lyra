# Lyra roadmap

**Hierarchy:**
- **Epic** `E<n>`: a capability.
- **Story** `E<n>.<s>`: a user- or developer-visible outcome.
- **Task** `E<n>.<s>.<t>`: a coherent piece of work.
- **Subtask** `E<n>.<s>.<t>.<a>`: **exactly one PR into `main`**.

Status marks: `[ ]` todo · `[~]` in progress · `[x]` merged (with PR number).

**Planning is rolling-wave.** The current and next epics are detailed down to subtasks. Later epics list their stories
and are refined into subtasks when the previous epic closes, because spike results change them. Every epic opens with
the spikes that could overturn it (`docs/design/spikes.md`). A spike's PR adds its report to `docs/spikes/`.

**GitHub mirror:**
- One issue per epic, story and subtask, linked as sub-issues.
- Each epic has a milestone.
- Labels: `type:epic|story|task|subtask|spike`, `area:<crate>`, `os:<windows|macos|linux>`, `security-critical`.

**Spend rule:** anything billable is quoted to the owner first. Model tests use the Claude API or a RunPod
Qwen3.8-27B endpoint (`docs/design/spikes.md`). No paid code signing until the owner approves it (E11).

## Phases

1. **Now: foundation + UX/UI.** E0 sets up the repository. E1 delivers the complete UX/UI as a working front-end
   prototype on a mock backend that speaks the planned protocol shapes, so the real core can replace it later.
2. **Paused after E1.** Core, backend and AI work (E2 onward) is **deferred until there is more time and funding**. The
   plan below stays as the specification for that work.

## Epic order and gates

| Epic | Outcome | Gate to start | Status |
|---|---|---|---|
| E0 Foundation | Repo, governance, README, repo settings, docs CI | — | active |
| E1 UX/UI | Design system, every v1 flow designed and built as a clickable front-end prototype on a mock backend; accessibility and visual-regression tests | E0 | next |
| E2 Core skeleton | Rust core + protocol + encrypted event log + hardened Electron shell, native approvals | E1, time and funding | deferred |
| E3 Vault and identity | Envelope vault on 3 OSes, assurance levels with user guidance, peer auth | E2 | deferred |
| E4 Providers and chat | Own adapters for 4 wire formats; streaming chat on any endpoint | E3, S-CORE1 | deferred |
| E5 Policy engine | Deterministic permission engine + shell command parser | E2 | deferred |
| E6 Sandbox | `lyra-sandbox`: Seatbelt / bwrap+seccomp / MXC + preflight | E5, sandbox spikes | deferred |
| E7 Egress proxy | `lyra-egress`: name-constrained CA, SSRF guard, audit | E6 | deferred |
| E8 Coding agent | Loop, durable state machine, tools, git `transmit`, checkpoints | E4, E6, E7 | deferred |
| E9 MCP and connectors | rmcp client, OAuth/CIMD, GitHub first | E8 | deferred |
| E10 Browser agent | Playwright executor in main, login handoff | E8 | deferred |
| E11 Computer use | macOS (Swift), Windows (Rust), Linux (Rust, flagged) | E8 | deferred |
| E12 Packaging and release | DMG/ZIP, MSIX/MSI, deb/rpm, signed manifest; signing once approved | E2 (unsigned), owner approval (signed) | deferred |
| E13 Evals and observability | promptfoo/Inspect via headless core, OTel, opt-in export | E4 | deferred |
| E14 VM tier | libkrun via microsandbox | E6, S-VM | deferred |
| E15 External engines | ACP adapters | E8 | deferred |
| E16 Remote clients | Core server mode, pairing, web client | E3, E8 | deferred |

---

## E0 Foundation

### E0.1 Repository and governance
- **E0.1.1 Bootstrap**
  - [x] E0.1.1.a Initial commit: licence, notice, governance files, README, `CLAUDE.md`, plan, design docs, ADRs.
    This is the only direct commit to `main`; everything after it goes through a PR.
- **E0.1.2 GitHub settings** (gh API; no PR)
  - [x] E0.1.2.a Repo settings: description, topics, squash-merge only, delete branch on merge, private vulnerability
    reporting, secret scanning + push protection, Dependabot alerts.
  - [x] E0.1.2.b Ruleset on `main`: PR required, linear history, no force-push or deletion.
  - [x] E0.1.2.c Labels, plus a milestone and an issue for each epic (stories and subtasks get issues when their epic starts).

### E0.2 Docs CI
- [x] E0.2.1.a `docs.yml`: Markdown link check, gitleaks, actionlint + zizmor on workflows (SHA-pinned actions,
  read-only token). Becomes a required check.

**E0 exit:** `main` is protected, and every PR runs the docs and secret checks.

---

## E1 UX/UI (next)

Built in `packages/ui` (React 19, Tailwind 4, Base UI, per `docs/design/stack.md`), with a mock backend in
`packages/mock-core` that returns fixtures shaped like the protocol in `docs/design/architecture.md`. The prototype
runs in a browser (Vite) and later inside the Electron shell unchanged. Detailed into subtasks when E1 starts.

- E1.1 Front-end toolchain and CI: pnpm workspace, TypeScript 7, oxlint, oxfmt, Vitest, Stryker (mutation), Playwright
  (visual regression + axe accessibility), Dependabot npm; a CI job on 3 OSes.
- E1.2 Design foundations: design tokens (colour, type, spacing, radius, motion), light, dark and high-contrast
  themes, icon set, WCAG 2.2 AA targets, per-OS conventions (window chrome, menus, shortcuts).
- E1.3 Component library + catalogue (Storybook or Ladle): primitives on Base UI, streaming message view, code/diff
  view, terminal frame, approval card, consent card, assurance-level badge.
- E1.4 Information architecture and app shell: navigation, threads, workspaces, command palette, settings.
- E1.5 Flows, each as a clickable prototype on mock data:
  - first run and model endpoint setup (any provider, self-hosted, gateway);
  - chat (multi-model, streaming, attachments, search);
  - coding agent workspace (plan, diffs, terminal, checkpoints, sandbox tier indicator);
  - approvals (native-dialog designs per OS: `transmit`, `destructive`, unsandboxed command);
  - browser agent (agent view, login handoff, confirmation cards);
  - computer use (allowed apps, consent, live overlay, Stop / kill switch, per-OS tier);
  - connectors (MCP catalogue, OAuth consent, tool classes);
  - vault and security (assurance level per OS with plain-language guidance and how to raise it; egress catalogue; audit log);
  - empty, error and offline states throughout.
- E1.6 Usability pass: keyboard-only and screen-reader walkthroughs, a review of every flow with the owner, and fixes.

**E1 exit:** every v1 flow is approved by the owner in the prototype, with axe checks and visual snapshots green on 3
OSes. Then the project pauses until E2 can be funded.

---

## E2 Core skeleton (deferred)

### E2.0 Rust workspace and core CI (moved from E0; first story when core work starts)
- [ ] E2.0.1.a Cargo workspace (edition 2024), `rust-toolchain.toml` (current stable at the time), workspace lints
  (`unsafe_code`, `unwrap_used`, `expect_used`, a clippy pedantic subset), `rustfmt.toml`, `.cargo/config.toml` (path
  remapping), and an `xtask` crate with `doctor`, `fmt`, `lint` and `test`.
- [ ] E2.0.1.b `xtask cov`: cargo-llvm-cov with a line-coverage floor (85%, ratchet up only).
- [ ] E2.0.1.c `xtask mutants --diff`: cargo-mutants on the diff vs `origin/main`; zero surviving mutants.
- [ ] E2.0.1.d `xtask ci` + `xtask docker-ci` (the Linux job in a digest-pinned image).
- [ ] E2.0.2.a `ci-rust.yml`: fmt, clippy, nextest, coverage on ubuntu/macos/windows; a sharded mutants job.
- [ ] E2.0.2.b `deny.toml` + cargo-audit, CodeQL for Rust, Dependabot cargo with a 7-day cooldown, a lockfile-age check.

### E2.1 Gating spikes
- [ ] E2.1.1.a Spike S-CORE3: build-loop budget on a scaffold workspace with the real dependency set (`docs/spikes/S-CORE3.md`).
- [ ] E2.1.2.a Spike S-CORE4: relay latency through Electron main (after E2.5.2).

### E2.2 Protocol (`lyra-protocol`)
- **E2.2.1 JSON-RPC**
  - [ ] E2.2.1.a Envelope types (request, response, notification, error codes), strict deserialization (deny unknown
    fields), and a size limit. Property tests: round-trip, and malformed input never panics.
  - [ ] E2.2.1.b Framing over byte streams (newline-delimited, max frame size), a dispatcher with a method registry,
    and cancellation.
- **E2.2.2 Domain model**
  - [ ] E2.2.2.a Thread / turn / item types, ids, and versioning (`protocolVersion`, capability negotiation).
  - [ ] E2.2.2.b Actuator (reverse-RPC) types: deadlines, pause leases, fail-closed semantics.
- **E2.2.3 Code generation**
  - [ ] E2.2.3.a `xtask gen`: `schemars` → JSON Schema → `ts-rs` types in `packages/protocol-ts`; a CI drift check.

### E2.3 Core binary (`lyra-core`)
- [ ] E2.3.1.a stdio server: launch-key hello on an inherited channel, `initialize`/`shutdown`, graceful exit on EOF.
- [ ] E2.3.1.b Per-OS data/config/log dirs with owner-only permissions/DACL, and refusal of unsafe parents.
- [ ] E2.3.1.c `tracing` setup: JSON logs to file, a redaction layer interface (implemented in E3), 14-day retention.

### E2.4 Event log and DB (`lyra-db`)
- [ ] E2.4.1.a rusqlite + bundled SQLCipher (spike S-DB folded in: crypto provider on MSVC/arm64; FTS5 present), and a
  `KeyProvider` trait (a test provider now; the vault in E3).
- [ ] E2.4.1.b Migrations framework + schema v1 (workspaces, threads, append-only `events`), with append-only
  enforced by triggers.
- [ ] E2.4.1.c FTS5 index over items, plus a redaction-on-write hook interface.
- [ ] E2.4.2.a Protocol methods: `thread/create|list|read`, `turn/start` (echo model), event streaming.

### E2.5 Desktop shell (`apps/desktop`; the UI comes from E1)
- [ ] E2.5.1.a Electron 45 app with the full hardening checklist (fuses, sandbox, CSP, `app://`, no `file://`), and a
  CI fuse-assertion test.
- [ ] E2.5.2.a Core launcher in main (spawn, inherited stdio, launch key, abort on any spawn or pipe error), and a frame relay.
- [ ] E2.5.2.b Typed IPC registry (one declarative table → preload bindings + main validators), rejecting non-top frames.
- [ ] E2.5.3.a Swap the E1 mock backend for the real protocol client (same contract), and the echo chat end to end.
- [ ] E2.5.4.a Playwright E2E smoke against the unsigned test build (inspect fuse on only in the test build), on 3 OSes.

### E2.6 Native approvals
- [ ] E2.6.1.a Main-process native approval dialog + single-use nonce minting. Core verifies the nonce on `transmit`
  actions; the renderer cannot approve (E2E test L-35).

### E2.7 Nightly unsigned builds (CD)
- [ ] E2.7.1.a Workflow: Forge make on 3 OSes (DMG/ZIP, MSIX unsigned/test cert, deb/rpm), with artifacts kept 14 days,
  SBOM (Syft) and checksums.

**E2 exit:** the unsigned nightly opens on 3 OSes, the echo chat works end to end, and fuse, CSP, nonce and DB-encryption
tests pass. One reviewer pass.

---

## E3 Vault and identity (security-critical)
- E3.1 Spikes: S-VAULT-W1/W3/W4, S-VAULT-L1/L2/L3, S-VAULT-V2 (AEAD interop) (`docs/design/spikes.md`).
- E3.2 `lyra-vault` envelope: VMK, XChaCha20-Poly1305, AAD scope binding, rotation, `KeyProvider` for the DB.
- E3.3 macOS custody. Dev builds: login keychain item bound to the binary (**L1**, labelled). Signed builds: the
  data-protection keychain and access group (**L3**), enabled in E12.
- E3.4 Windows custody: TPM-bound CNG key (**L2**); DPAPI (**L1**) only with explicit opt-in.
- E3.5 Linux custody: `systemd-creds --user` + TPM2 where systemd ≥ 256 (**L2**), else Secret Service (**L1**); refuse if none.
- E3.6 `lyra-harden`: Windows mitigation policies + `OWNER RIGHTS` DACL; Linux non-dumpable, Yama check, `mlock`.
- E3.7 `lyra-peer`: macOS audit token, Windows pipe DACL + Authenticode, Linux `SO_PEERPIDFD` / `SO_PEERCRED`.
- E3.8 Vault protocol (`vault/put|list|delete|rotate`, no get) and the UI: **plain-language assurance guidance per level
  and OS**, with what the user can do to raise it.
- E3.9 Redaction: keyed fingerprints and gitleaks-derived rules; the redaction layer wired into logs, DB and tool output.
- E3.10 Security acceptance tests (`docs/design/secrets.md`) on 3 OSes.

## E4 Providers and chat
- E4.1 Spike S-CORE1 + S-AF6: provider parity on 20 tasks (**quote Claude API / RunPod cost first**).
- E4.2 Provider interface, SSE client, OpenAI Chat-compatible adapter (vLLM, SGLang, LiteLLM, xAI, DeepSeek, gateway).
- E4.3 Anthropic Messages adapter (thinking replay, caching, computer-use tool format).
- E4.4 OpenAI Responses adapter.
- E4.5 Gemini `generateContent` adapter.
- E4.6 Endpoints: records, capability probe, tier aliases, host pinning, body allow-list, no cross-host redirects.
- E4.7 Chat UI: Streamdown (locked down), threads, search, streaming render budget (S-UI-1..3).
- E4.8 Recorded-fixture contract tests per provider; `genai` differential oracle.

## E5 Policy engine
- E5.1 Policy model: action classes (read, write, transmit, destructive, financial, permission), narrow-only layering.
- E5.2 Shell parser on tree-sitter-bash: fail closed on parse errors; property tests; CI cross-check corpus.
- E5.3 Decision engine + audit log; deterministic, no LLM judge in v1.

## E6 Sandbox (security-critical)
- E6.1 Spikes: S-M1/M2 (macOS), S-L1/L2 (Linux), S-W1/W2/W3 (Windows; **needs a Windows 11 machine or VM**).
- E6.2 Policy compiler, path resolver, credential-path audit, mandatory deny lists (R1–R2).
- E6.3 Launcher: env scrub, limits, watchdog, process-tree kill, pty/stdio (R6–R7).
- E6.4 macOS backend: SBPL generator + `sandbox-exec`.
- E6.5 Linux backend: bundled bwrap, seccomp, netns + Unix-socket bridge, Landlock second layer, AppArmor profile.
- E6.6 Windows backend: `mxc-sdk` ProcessContainer + outer job; refuses any tier but `base-container`.
- E6.7 Preflight probe suite (R9) on every launch and in CI on 3 OSes.
- E6.8 File-tool worker mode.

## E7 Egress proxy (security-critical)
- E7.1 Spike S-E1.
- E7.2 CONNECT + SNI checks, domain matcher, resolved-address guard (SSRF), DNS by the proxy.
- E7.3 Name-constrained in-memory CA; MITM only for method/path-ruled hosts.
- E7.4 Session tokens, revocation at kill, audit log, egress catalogue enforcement.

## E8 Coding agent
- E8.1 Spike S-AF1 (durability under kill -9).
- E8.2 Agent loop, subagents, compaction.
- E8.3 Durable turn state machine over the event log (no automatic retry of gated actions).
- E8.4 File and shell tools through `lyra-sandbox`; output redaction.
- E8.5 `transmit` actions: git push and publish performed by core after native approval.
- E8.6 Checkpoints, diff review UI (CodeMirror merge), terminal (xterm in its own CSP frame).
- E8.7 Import of `AGENTS.md` / `CLAUDE.md` / `GEMINI.md` and skills.

## E9 MCP and connectors
- E9.1 rmcp client (stdio under the sandbox, streamable HTTP), schema-hash pinning, exact-name tool classes.
- E9.2 OAuth 2.1 + PKCE, RFC 9728, CIMD, tokens in the vault.
- E9.3 Connectors: GitHub first, then Atlassian, Slack, Figma, Google Drive, Telegram; Teams last.
- E9.4 Config import from other apps (moving secrets into the vault).

## E10 Browser agent
- E10.1 Spikes S3a–S3d.
- E10.2 Executor in main (Playwright over `webContents.debugger`; Chrome/Edge pipe fallback).
- E10.3 Login handoff, narrow tools, confirmation cards, sensitive-page classes.

## E11 Computer use
- E11.1 `lyra-cu-proto` + conformance harness (S-X).
- E11.2 macOS Swift helper (S5).
- E11.3 Windows Rust helper (S-Win-1..3).
- E11.4 Linux Rust helper, tiered, behind a flag (S-Lin-1..3).

## E12 Packaging and release
- E12.1 Own Ed25519 update manifest (anti-rollback), Squirrel.Mac / MSIX update, apt/dnf repos.
- E12.2 Linux AppArmor profiles, root-owned install, signed hash-list check at launch.
- E12.3 **Signing (owner approval and spend required):** Apple Developer ID + notarization (enables vault L3 on macOS),
  Windows Artifact Signing or OV.
- E12.4 Reproducible builds and an SBOM per release.

## E13 Evals and observability
- E13.1 OTel GenAI spans from the event log; opt-in OTLP export (Langfuse documented); zero-egress test (S-AF3).
- E13.2 promptfoo + Inspect AI through headless core in CI (S-AF4).
- E13.3 Prompt-injection corpus (AgentDojo/InjecAgent subsets), garak/PyRIT in dev.

## E14 VM tier
- E14.1 Spike S-VM; microsandbox/libkrun executor on a workspace copy; egress via `lyra-egress` only.

## E15 External engines
- E15.1 ACP client; adapters for Gemini CLI, agy, Claude Code (`claude-agent-acp`), Codex (`codex-acp`).

## E16 Remote clients
- E16.1 Core server mode, device pairing, web client.
