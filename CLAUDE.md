# CLAUDE.md — working context for Lyra

Lyra is an open-source (Apache-2.0), provider-neutral desktop AI app for Windows, macOS and Linux. It covers chat across
model providers, a sandboxed coding agent, an agent-controlled browser, computer use and MCP connectors. It has a Rust
core (`lyra-core`), an Electron shell (UI and browser host), and its own sandbox, egress proxy and vault.

Read this file first in every session. It is kept current: **any PR that adds, moves or removes a crate, package,
module, binary, script or workflow updates the repo map below in the same PR.**

## 0. Current phase

**E1 UX/UI**: front-end prototype on a mock backend. Core, backend and AI work (E2+) is deferred until there is time
and funding. Do not start Rust crates unless the owner reopens E2.

## 1. Where things are decided

| Question | Source of truth |
|---|---|
| What to build next, and its status | `docs/plan/ROADMAP.md` (epics → stories → tasks → subtasks; one subtask = one PR). GitHub issues mirror it |
| How a subsystem must work | `docs/design/` (spec). Start with `docs/design/README.md` |
| Why a choice was made | `docs/adr/` (one ADR per decision; supersede, never edit history) |
| Exact dependency versions | `docs/design/stack.md` and the lockfiles; check the `dependency-registry` skill before adding or upgrading |
| Spikes and kill criteria | `docs/design/spikes.md`; results go in `docs/spikes/<id>.md` |
| Session state (local only, gitignored) | `STATE.md` |

The private research behind the design lives outside this repo (`../lyra-research`, private GitHub repo). Never copy
its content here: it holds unverified findings about third-party closed-source apps.

## 2. Repo map

Status: **present** = exists on `main`; **planned** = reserved name, created by the subtask in brackets. Before creating
anything, search for an existing home (`rg -n "<name>"`, `cargo tree`, and this map). Do not create a parallel module,
helper or util file when one exists. Extend the existing one, or move it in a dedicated refactor PR.

```
lyra/
  CLAUDE.md  AGENTS.md  README.md  LICENSE  NOTICE  CONTRIBUTING.md  SECURITY.md  CODE_OF_CONDUCT.md   present
  .github/                      issue/PR templates, CODEOWNERS; workflows/docs.yml (links, secrets, public-content, workflows)   present
  docs/design/ docs/adr/ docs/plan/                                                   present
  docs/spikes/                  spike reports                                         planned [first spike]
  Cargo.toml rust-toolchain.toml deny.toml rustfmt.toml .cargo/                       planned [E2.0.1.a]
  xtask/                        `cargo xtask <ci|fmt|lint|test|cov|mutants|docker-ci|gen>`  planned [E2.0.1.a]
  crates/lyra-protocol/         JSON-RPC types, thread/turn/item model, schema + TS generation   planned [E2.2]
  crates/lyra-core/             core binary: stdio server, session, agent loop, durable state machine   planned [E2.3]
  crates/lyra-db/               rusqlite + SQLCipher, migrations, event log, FTS5     planned [E2.4]
  crates/lyra-vault/            envelope vault, per-OS VMK custody, hardening         planned [E3]
  crates/lyra-peer/             per-OS IPC peer authentication                        planned [E3]
  crates/lyra-providers/        model adapters (4 wire formats)                       planned [E4]
  crates/lyra-policy/           permission engine, tree-sitter-bash command parser    planned [E5]
  crates/lyra-sandbox/          policy compiler + Seatbelt / bwrap / MXC backends     planned [E6]
  crates/lyra-egress/           egress proxy binary                                   planned [E7]
  crates/lyra-mcp/              rmcp client wrapper, registry, OAuth                  planned [E9]
  crates/lyra-cu-proto/         computer-use contract + shared policy                 planned [E11]
  helpers/                      LyraComputerUse (Swift), lyra-cu-windows, lyra-cu-linux   planned [E11]
  packages/ui/                  React UI, design system, component catalogue         planned [E1.1]
  packages/mock-core/           mock backend returning protocol-shaped fixtures      planned [E1.1]
  apps/desktop/                 Electron main + preload, fuses, packaging             planned [E2.5]
  packages/protocol-ts/         generated TS protocol types (never hand-edit)         planned [E2.2]
```

## 3. Non-negotiable invariants (security)

Breaking any of these is a release blocker. Each has a test id in `docs/design/` that must stay green.

1. **The renderer is untrusted.** It can never approve an action. Approvals are native dialogs owned by Electron main,
   which carry a single-use nonce that core verifies.
2. **Secrets live only in the vault.** No `vault/get` over the protocol. No secret in argv, env of long-lived processes,
   logs, transcripts, the DB in clear, config files, URLs, or model context. **No plaintext fallback, ever.**
3. **Sandboxed code holds no credentials.** Authenticated side effects (git push, publish) are `transmit` actions that
   core performs after native approval.
4. **Default-deny everywhere:** sandbox reads and writes, network (only via `lyra-egress`), MCP tool classes, and egress
   destinations (catalogue). A project config can only narrow policy, never widen it.
5. **Every tool call goes through Lyra's loop, policy engine and sandbox.** Never adopt a framework or SDK feature that
   executes tools itself.
6. **Zero egress by default.** Telemetry export is opt-in. Lyra makes no vendor calls of its own.
7. **Parity or disclosure:** a control that is weaker on one OS is shown to the user (assurance level, tier). It is
   never silently downgraded.

## 4. How work is done

- **Unit of work = one subtask = one PR into `main`.** Branch `e<N>/<id>-<slug>`, e.g. `e1/1.3.2.a-approval-card`.
  PR title `[E2.2.1.a] <imperative summary>`. Body: `Closes #<issue>`, what changed, how it was tested, risks.
- Commits follow Conventional Commits (`feat(core): …`, `fix(vault): …`, `test:`, `docs:`, `ci:`, `chore:`). Sign
  off with DCO (`git commit -s`). Squash-merge only. Linear history.
- **Definition of done** (CI enforces it):
  1. `cargo xtask ci` passes locally, and the Linux job passes in Docker (`cargo xtask docker-ci`).
  2. CI is green on Linux, macOS and Windows.
  3. Every changed line of logic is covered by tests, and **mutation testing on the diff leaves zero surviving mutants**
     (`cargo xtask mutants --diff`; Stryker for TS once it exists). A `#[mutants::skip]` needs a comment saying why.
  4. Parsers, policy and path logic have property tests (`proptest`).
  5. `docs/plan/ROADMAP.md` ticks the subtask; this file's repo map is updated if the layout changed; an ADR is added
     if a decision changed.
- Security-critical stories (vault, peer, policy, sandbox, egress, approvals) end with one independent `reviewer` pass.
  Other epics get one at the end.
- Never commit secrets, `.env` files, research content, or generated files other than `packages/protocol-ts`
  (CI checks that it matches the generator).

## 5. Commands

| Command | Does |
|---|---|
| `cargo xtask ci` | Everything CI runs that works locally: fmt check, clippy `-D warnings`, nextest, coverage floor, deny, audit |
| `cargo xtask mutants --diff` | cargo-mutants on lines changed vs `origin/main` |
| `cargo xtask docker-ci` | The Linux CI job in the pinned Docker image |
| `cargo xtask gen` | Regenerate JSON Schema + TS protocol types (planned [E2.2]) |

Local tools (not project dependencies): `cargo-nextest`, `cargo-llvm-cov`, `cargo-deny`, `cargo-audit`,
`cargo-mutants`, Docker. `cargo xtask doctor` reports what is missing.

## 6. Conventions

- Rust: edition 2024, toolchain pinned in `rust-toolchain.toml`. `unsafe` is denied workspace-wide except in modules
  named `sys` (FFI), each with a `// SAFETY:` comment per block. Errors via `thiserror` in libraries and `anyhow` only
  in binaries and xtask. No `unwrap()`/`expect()` outside tests (clippy enforces this).
- Logging via `tracing`. Never log secret material; redaction happens before `tracing` sees the value.
- Dependencies: crates.io only (no git dependencies). New crates are checked against `deny.toml` and must be at least 7
  days old. Prefer the standard library and existing workspace dependencies.
- Identifiers: `dev.thehenrylabs.lyra` (app), `dev.thehenrylabs.lyra.core`, `dev.thehenrylabs.lyra.computeruse`.
  Vault data is bound to them. Never change them.
- Docs: plain, specific English; tables for comparisons; no marketing.
