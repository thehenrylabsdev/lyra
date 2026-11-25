# Architecture

Status: accepted design for v1. The key words MUST, MUST NOT, SHOULD and MAY are used as in RFC 2119.
Decisions are recorded in `docs/adr/`. Exact versions are in [stack.md](stack.md).

## 1. Goals and non-goals

Goals:
- One desktop app for chat with any model, a sandboxed coding agent, an agent-controlled browser the user logs
  into, computer use, and MCP integrations, on Windows, macOS and Linux (x64 and arm64).
- **Provider-neutral.** Every model call goes from the user's machine to an endpoint the user configured: a vendor
  API, a gateway (LiteLLM, Vercel AI Gateway) or a self-hosted server (vLLM, SGLang). There is no Lyra relay.
- **Secure by construction.** Secrets never reach the model, the sandbox, a config file, a log, argv or a URL. Every
  executor is confined. Every outbound destination is named in the egress catalogue (§9).
- Interoperable: Lyra can drive external agent engines over the Agent Client Protocol (ACP), each with its own login.

Non-goals for v1: a Lyra cloud service, hosted MCP placement, a model marketplace, mobile clients, and any use of
consumer-app sessions in place of official APIs.

## 2. Process map

```
renderer (React UI)                       UNTRUSTED: renders model output; cannot approve anything
   │  typed IPC registry (preload, validated, origin-tagged)
   ▼
Electron main (TypeScript)                relay · native approval dialogs + nonce minter · browser executor
   │                                      (playwright-core over webContents.debugger) · core launcher · updater
   │  inherited stdio, JSON-RPC 2.0 (newline-delimited)
   ▼
lyra-core (Rust, separately signed)       agent loop · durable state machine · providers · MCP/ACP clients ·
   │                                      envelope vault · encrypted DB · policy engine · telemetry projection
   ├─ spawns (inherited fds) ──► lyra-sandbox (one per command) ──► lyra-egress (one per session) ──► allowed hosts
   ├─ named socket/pipe + peer auth + pairing nonce ──► computer-use helper (one per OS)
   ├─ actuator calls browser/* ──► Electron main (webContents.debugger exists only there)
   ├─ ACP over stdio ──► external agent engines (opt-in)
   └─ T2 only ──► VM tier (libkrun via microsandbox, after spike S-VM)
```

| Component | Language | Identity on each OS |
|---|---|---|
| Electron app | TS | macOS `dev.thehenrylabs.lyra`; Windows MSIX package; Linux `/opt/Lyra` (root-owned) |
| `lyra-core` | Rust | macOS `Lyra.app/Contents/Helpers/LyraCore.app`, bundle id `dev.thehenrylabs.lyra.core`, own provisioning profile and keychain group `TEAMID.dev.thehenrylabs.lyra.vault`; Windows `lyra-core.exe` inside the MSIX; Linux `/opt/Lyra/resources/bin/lyra-core` |
| `lyra-sandbox` | Rust | Multi-call binary next to core (launcher and file-tool worker modes) |
| `lyra-egress` | Rust | Separate binary; on Windows it runs in its own AppContainer profile (spike S-W2) |
| Computer-use helper | Swift (macOS), Rust (Windows, Linux) | macOS `LyraComputerUse.app`, `dev.thehenrylabs.lyra.computeruse` (own TCC identity); Windows `lyra-cu-windows.exe`; Linux `lyra-cu-linux` |
| CLI (later) | Rust | Own signing identifier `dev.thehenrylabs.lyra.cli`; never borrows the app's keychain or TCC trust |

Why this shape: the code that holds secrets and decides policy is one Rust binary with its own code identity and no
JIT, and the sandbox, egress, vault and policy crates link into it directly. Electron stays as the UI and browser
host because it is the only shell with the same renderer sandbox, CSP and in-process CDP on all three OSes
(ADR 0001, ADR 0002). The pattern of a Rust core under an Electron UI, speaking JSON-RPC over stdio, follows the
open-source [`openai/codex`](https://github.com/openai/codex) app-server.

## 3. Trust boundaries

| Id | Boundary | Rule |
|---|---|---|
| TB-1 | Renderer → main | The renderer is untrusted. It reaches only methods in the typed IPC registry. Main tags every relayed frame with origin `renderer`. It MUST NOT be able to approve, widen policy, add endpoints or servers, or touch the vault |
| TB-2 | Main ↔ core | Inherited stdio only; no listening socket or port. The first frame is `hello{protocolVersion, buildId, launchKey}`. Core refuses a mismatched `buildId`. `launchKey` (32 random bytes, never on argv or in env) keys the approval HMAC (§4) |
| TB-3 | Core → sandboxed children | Children are untrusted. They get an allowlisted environment, no secrets (except an MCP server's mapped secrets), no keystore, DB, socket or pipe access, and network only through `lyra-egress` |
| TB-4 | Sandbox → `lyra-egress` | The proxy parses attacker-controlled traffic. It runs outside the sandbox as its own process, one per session, with a per-session token that core revokes when the session's processes are killed |
| TB-5 | Core ↔ computer-use helper | The helper is privileged (screen, input). It accepts commands only from an authenticated core (peer auth per OS plus a per-launch pairing nonce; [secrets.md](secrets.md) §6) |
| TB-6 | Agent browser content | Untrusted web content runs in a `WebContentsView` with its own session partition and no preload. The model never sees cookies, storage, raw CDP or JS evaluation |
| TB-7 | External engines (ACP) | Untrusted. They keep their own credentials; Lyra renders their items and enforces its approval UI where the engine delegates approvals |

A compromised main process is a full compromise; main is kept small (about 1–2k LOC TS) to keep that surface reviewable.

## 4. Native approval dialog with nonce

**Trusted decisions.** These MUST be confirmed in a native dialog owned by the main process:
1. Approving an action of class `transmit`, `destructive`, `financial` or `permission` ([sandbox.md](sandbox.md) §7).
2. Running a T0 (unsandboxed) command.
3. Any policy widening (new domain, path root or allow rule), including a sandbox escalation request.
4. Adding a model endpoint, an MCP server, an OTLP export endpoint or a connector.
5. Changing a secret's allowed consumers, or lowering a secret kind's minimum vault level.

**Flow.**
1. Core emits `approval/requested{requestId, threadId, itemId, actionClass, summary, actionHash}`. `actionHash` is
   SHA-256 over the canonical JSON of the exact action (command argv and cwd, host and method, recipient and text, policy diff).
2. Main renders a native dialog (`dialog.showMessageBox`, modal to the main window; on Wayland it MUST be modal rather
   than relying on always-on-top, spike S-SH2). The dialog shows the exact action, not a model-written summary alone.
3. On a decision, main computes `token = HMAC-SHA256(launchKey, requestId ‖ threadId ‖ itemId ‖ actionHash ‖ decision)`
   and sends `approval/respond{requestId, decision, token}` with origin `main`.
4. Core accepts only if the request is pending, the token verifies (constant-time compare), and the token was not used
   before. The token is single-use. The request expires when its turn ends or is interrupted.

Invariants:
- A message from the renderer is never sufficient. Core rejects `approval/respond`, `config/patch` (widening),
  `mcp/add`, `provider/add`, `vault/*` and `export/add` without a valid main-minted token (threat L-35; test in §6).
- No agent tool can call `approval/*`, `config/*`, `mcp/*` or `vault/*`.
- Rich context (a diff, a commit list) MAY be shown in the UI for review, but the decision is captured only by the
  native dialog, which restates the action and the first 8 hex digits of `actionHash`.

## 5. Protocol

JSON-RPC 2.0, schema-first. Rust types in `lyra-protocol` generate JSON Schema (`schemars`), which generates the
TypeScript types (`ts-rs`) in `packages/protocol-ts`. CI fails on drift. Framing is one JSON object per line over
stdio; writers batch and flush on idle. Unknown fields are rejected. A maximum frame size is enforced.

**Objects.** `Workspace` → `Thread` → `Turn` → `Item`. Item kinds: `user_message`, `assistant_message`, `reasoning`,
`tool_call`, `tool_result`, `approval_request`, `approval_decision`, `file_change`, `artifact`, `policy_change`, `error`.
Every item is an event in the append-only log (§7); sandbox mode, approval policy and model choice are events too, so
resume and audit reproduce exactly what applied.

**Client → core.**

| Method | Notes |
|---|---|
| `thread/start`, `thread/resume`, `thread/fork`, `thread/archive`, `thread/list`, `thread/read` | |
| `turn/start`, `turn/interrupt` | |
| `approval/respond` | Needs a main-minted token (§4) |
| `config/get`, `config/patch` | Widening needs a token |
| `provider/list`, `provider/add`, `provider/test` | `provider/add` needs a token |
| `mcp/list`, `mcp/add`, `mcp/remove`, `mcp/authorize` | `mcp/add` needs a token |
| `vault/status`, `vault/list`, `vault/put`, `vault/delete`, `vault/rotate` | **There is no `vault/get`.** `vault/put` and `vault/delete` need a token |
| `browser/handoff/complete` | Sent by main after the user finishes a login handoff |

**Core → client notifications.** `item/started`, `item/delta`, `item/completed`, `turn/completed`,
`approval/requested`, `status/changed`, `usage/updated`, `vault/levelChanged`.

**Actuator protocol (reverse RPC, core → executor).** Methods `exec/run`, `fs/read`, `fs/write`, `fs/patch`,
`browser/*`, `computer/*`. Every call carries:
- `deadline` (absolute); an executor MUST abort at the deadline;
- `idempotencyKey`; an executor MUST answer a repeated key with the original result, not a second execution;
- `lease{id, expiresAt}`; core renews it while the turn runs. **An executor that loses its lease stops (fail-closed)**
  and reports `lease_lost`. Pausing a turn means not renewing the lease.

**Versioning.** `protocolVersion` is in every hello. Changes within a major version are additive only.

**Initial error codes.** `-32001 approval_required`, `-32002 token_invalid`, `-32003 sandbox_unsupported`,
`-32004 lease_lost`, `-32005 policy_denied`, `-32006 vault_level_insufficient`, `-32007 egress_denied`.

The same protocol serves the desktop (via main), the headless CLI (stdio) and later server mode (`lyra-core serve`
over WebSocket with device pairing). Executors always stay on the user's machine.

## 6. Durable turn state machine

Each tool call has a persisted lifecycle in the event log:

```
proposed → policy_evaluated → approval_requested → approval_decided(token) → dispatched(idempotencyKey, lease)
        → completed | failed | lease_lost | interrupted
```

Recovery on restart (core replays the log):
- A call left in `dispatched` is resolved by asking its executor about the idempotency key. Unknown → `interrupted`.
- `read`, `write-local` and `exec` calls in T1/T2 MAY be re-offered automatically. `transmit`, `destructive`,
  `financial` and `permission` calls are **never retried automatically**; they need a fresh approval.
- Model streams are not durable mid-stream; the step restarts from the last committed item.
- Scheduled or background runs use a `schedules` table and catch up at launch.

Acceptance: spike S-AF1 (kill -9 at 7 lifecycle points × 3 action classes on all three OSes) shows no duplicate
gated action, no lost approval and no write after lease expiry. Test L-35: a forged `approval/respond` without a
valid token is rejected.

## 7. Storage layout

| OS | Data directory | Notes |
|---|---|---|
| macOS | `~/Library/Application Support/Lyra/` | `0700`; files `0600`; excluded from Spotlight |
| Windows | `%LOCALAPPDATA%\Lyra\` | Owner-only DACL, inheritance disabled, `FILE_ATTRIBUTE_NOT_CONTENT_INDEXED` |
| Linux | `$XDG_DATA_HOME/lyra/` (default `~/.local/share/lyra/`); sockets in `$XDG_RUNTIME_DIR/lyra/` | `0700`/`0600`; refuse a world-writable parent |

```
<data dir>/
  lyra.db              one SQLCipher 4 database: workspaces, threads, append-only events, settings,
                       provider endpoints (secret references only), mcp_servers, policies, egress_catalog,
                       audit_log, schedules, FTS5 index over redacted text, vault records (AEAD ciphertext)
  vault.key            Windows only: TPM-wrapped VMK blob (owner-only DACL)
  artifacts/<id>/      content-addressed blobs
  checkpoints/         pre-edit snapshots; agent-unreadable and write-denied
  browser-profiles/<id>/  agent browser partitions (0700)
  logs/                redacted, size-capped, 14-day default retention
```

- The DB key is `HKDF-SHA256(VMK, "lyra/db/v1")`, passed as a raw 32-byte key; it is never stored ([secrets.md](secrets.md)).
- **Redaction on write:** a scanner (gitleaks-derived rules plus keyed fingerprints of vault values) runs before
  anything is persisted, indexed or logged.
- Checkpoints use a copy-on-write clone where the filesystem supports it (APFS `clonefile`, Linux `FICLONE` on
  Btrfs/XFS, ReFS block cloning), else a copy. Git repos also get a ref in a Lyra-private namespace.
- If the DB cannot be opened, Lyra starts in a labelled in-memory mode and never blocks the user.
- Lyra's own data directory is read- and write-denied to every sandboxed process and excluded from exports unless asked.

## 8. Configuration and policy layering

Order, later wins except that **deny always wins**: `defaults < user < workspace (project) < local < managed`.

- The managed layer is fail-closed.
- **The project layer can only narrow.** It MAY add `deny` or `ask` rules and deny-paths. It MUST NOT add `allow`
  rules, network domains, or read/write roots. A repo shipping `allow: Browser.transmit(*)` is ignored with a warning,
  and the widening is offered to the user as a native-dialog proposal.
- Project config MUST NOT set provider endpoints, base URLs, proxies, CA certificates, headers, provider env, hooks
  that run outside the sandbox, or auto-starting MCP servers. These need user-layer consent. (Some agent tools let a
  cloned repo redirect the model base URL; Lyra does not.)
- **Strict mode:** one switch that disables every optional risk (T0, browser `transmit`, computer use, untrusted connectors).
- Lyra's config, policy and plugin directories are write-denied to all agent tools.

## 9. Egress catalogue

Core keeps a catalogue of every destination Lyra itself may contact, with the feature and the reason. The UI renders
it, and core enforces it: a destination that is neither in the catalogue nor in user config is refused. Telemetry is
off by default, Lyra makes no vendor calls of its own, and there are no stable device ids.

| Destination | Feature | When | Who dials |
|---|---|---|---|
| User-configured model endpoints | Chat, agent, judge tier | On use | core (host-pinned) |
| User-added remote MCP servers and their OAuth authorization servers | Connectors | On use | core |
| Built-in connectors: `api.githubcopilot.com`, `github.com`, `mcp.atlassian.com`, `mcp.figma.com`, `drivemcp.googleapis.com`, `mcp.slack.com`, `graph.microsoft.com`, `login.microsoftonline.com`, `api.telegram.org` | Connectors ([mcp-and-connectors.md](mcp-and-connectors.md)) | Only after the user enables the connector | core |
| Lyra's Client ID Metadata Document URL (HTTPS, project-controlled) | MCP OAuth (CIMD) | Fetched by authorization servers, not by Lyra | — |
| Lyra update manifest host (project-controlled) | Update check | Daily, release builds only; off in dev builds | main |
| User-named OTLP endpoint | Trace export | Opt-in only; added through the native dialog | core |
| Sites the user browses in the agent browser | Browser agent | Per task origin allow-list | main (browser executor) |
| Sandbox default fetch-only allowlist: `registry.npmjs.org`, `pypi.org`, `files.pythonhosted.org`, `github.com` (fetch paths), `index.crates.io`, `static.crates.io` | Coding agent | Through `lyra-egress` only; editable per workspace | `lyra-egress` |

Fail-closed tests: on a fresh install with only a LAN vLLM endpoint configured, a network capture shows no other host
(threats L-26, L-27; spike S-AF3). Microsoft MXC is built with its telemetry unrouted, verified by an ETW check in CI.

## 10. Electron hardening checklist (release gate)

| # | Control | Check |
|---|---|---|
| H1 | Fuses: `RunAsNode` off, `EnableNodeOptionsEnvironmentVariable` off, `EnableNodeCliInspectArguments` off, `EnableCookieEncryption` on, `EnableEmbeddedAsarIntegrityValidation` on, `OnlyLoadAppFromAsar` on, `GrantFileProtocolExtraPrivileges` off, `WasmTrapHandlers` on | CI reads the fuses of every built artifact (`@electron/fuses` read) on all three OSes |
| H2 | E2E exception: a separate **test** build turns `EnableNodeCliInspectArguments` on for Playwright. It is never distributed; CI asserts both flavours | CI |
| H3 | All windows: `sandbox: true`, `contextIsolation: true`, `nodeIntegration: false`, `webviewTag: false` | Unit test over window factory; grep for `webviewTag` |
| H4 | Untrusted web content only in `WebContentsView` with its own session, no preload | Test |
| H5 | CSP: `default-src 'none'; script-src 'self'; style-src 'self'; style-src-elem 'self' 'nonce-<per-load>'; style-src-attr 'none'; img-src 'self' data: blob:; font-src 'self'; connect-src 'none'; worker-src 'self' blob:; object-src 'none'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'; require-trusted-types-for 'script'`. No `unsafe-eval`. The terminal runs in its own `app://terminal` frame, which alone may allow `style-src-elem 'unsafe-inline'` (spike S-UI-2) | CSP violation reporter in E2E; XSS corpus (S-UI-3) |
| H6 | Custom `app://` scheme; no `file://` loading | Test |
| H7 | Typed IPC registry: one declarative table generates preload bindings, main handlers and argument/return validators; non-top frames and opaque origins are rejected | Test |
| H8 | `setWindowOpenHandler` denies by default; `will-navigate` is blocked for app windows; permission request handlers deny by default | Test |
| H9 | DevTools disabled, no source maps, no internal or staging hosts, no test origins in release builds | Release audit script in CI, zero findings |
| H10 | macOS: ATS default (no `NSAllowsArbitraryLoads`); minimal entitlements on the main app; TCC-heavy capabilities only in helpers | Info.plist and entitlements check in CI |
| H11 | Linux: never `--no-sandbox`; .deb/.rpm install the AppArmor profile and `chrome-sandbox` helper; CI asserts a sandboxed renderer | CI on Ubuntu 24.04 |
| H12 | Deep links: schema-validated; every action confirmed; a link never auto-runs a prompt | Fuzz test |
| H13 | Updates: HTTPS only; Lyra Ed25519 manifest (version, monotonic sequence, expiry, hashes) verified in main; no hot-loaded remote code | Replay, downgrade, expired and truncated manifests refused (spike S-SC5) |
| H14 | Secret-bearing windows set `WDA_EXCLUDEFROMCAPTURE` on Windows | Test |
| H15 | Markdown from models goes AST → React elements; never `innerHTML` or `dangerouslySetInnerHTML`; no remote images | Lint rule + XSS corpus |
| H16 | Electron security releases ship within 7 days | Release policy |
