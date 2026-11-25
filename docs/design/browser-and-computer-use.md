# Browser agent and computer use

Status: accepted design for v1. Browser in phase "browser agent"; computer use ships macOS and Windows first, Linux
behind a flag until spike S-X passes.

## Part A: Browser agent

### A1. Pattern

A Lyra-owned, visible agent browser with a dedicated profile, controlled in-process and never through a TCP port.

| Option | Use |
|---|---|
| **A. Electron partition (default)** | `WebContentsView` with a dedicated persistent `session` partition, driven by `playwright-core` over `webContents.debugger` (CDP in-process). `webContents.debugger` exists only in the main process, so the browser executor lives in main and core drives it through `browser/*` actuator calls |
| **B. Stable Chrome (fallback)** | The user's installed stable Chrome (Windows: Chrome, else Edge; Linux: Chrome .deb/.rpm only) with a dedicated `--user-data-dir`. **Login runs in a launch without any debugging flag**; Lyra then relaunches the same profile with `--remote-debugging-pipe` (a pipe, not a port). For sites that reject embedded or automated browsers |
| C. Extension + native messaging into the user's own Chrome | Not in v1. Later opt-in only, with per-site grants and no `debugger` permission |
| D. Remote/cloud browser | Not in v1 |

Public constraints this design follows:
- Since Chrome 136, `--remote-debugging-port` and `--remote-debugging-pipe` are ignored for the default data
  directory; a non-default `--user-data-dir` is required and uses a different encryption key
  ([Chrome blog](https://developer.chrome.com/blog/remote-debugging-port)). Lyra always uses a dedicated directory, for Edge too.
- Administrators can disable remote debugging with the `RemoteDebuggingAllowed` policy
  ([Edge policy docs](https://learn.microsoft.com/deployedge/microsoft-edge-browser-policies/remotedebuggingallowed)).
  Lyra probes the policy (registry, `/etc/opt/chrome/policies`, managed preferences) and falls back to option A with a
  visible message. It never tries to bypass policy.
- Lyra does **not** mask automation signals (e.g. `AutomationControlled`) by default; at most it is an explicit
  per-site user choice (threat L-33).

### A2. Login handoff

1. The agent calls `browser.request_login({origin, reason})`.
2. The UI shows the registrable domain prominently and refuses domains not on the task's origin list (anti-phishing).
3. The pane switches to **user control**: agent input is disabled, and agent screenshots and accessibility reads stop.
4. The user signs in with their own password manager, passkey or 2FA. Lyra never sees credentials.
5. The user clicks **Done**. Main checks the login (URL pattern or cookie **presence**; the model never reads cookies)
   and sends `browser/handoff/complete` to core, which resumes the agent.
6. CAPTCHAs, password changes and payment entry are always handoff.
7. **Model-blind fill is deferred.** When built: the user stores a site credential in the vault; a native sheet shows
   the origin; the main-process injector picks the target fields itself (`autocomplete="username|current-password"`,
   `type=password`, top-level frame of the approved origin) and returns only `success|failure`. The model's element
   reference never decides where a password goes. Fill is disabled if any debugging transport is externally reachable.

### A3. Tool API (narrow by design)

`navigate(url)` · `snapshot()` (accessibility tree with element refs; hidden, zero-size and off-screen text stripped;
size-capped) · `screenshot()` (password and OTP fields masked) · `click(ref)` · `type(ref, text)` · `select(ref, value)` ·
`scroll` · `wait_for` · `upload(ref)` (opens a **user** file picker; no arbitrary paths) · `download()` (to a quarantine
dir) · `tabs.list|switch|close` · `request_login` · `extract(schema)`.

**Never exposed to the model:** raw CDP, `Runtime.evaluate` or any JS evaluation, cookies (`Network.getCookies`,
`Storage.*`, `document.cookie`), local or session storage, `Fetch.*` interception, and response headers carrying
`Set-Cookie` or `Authorization`. Navigation to `file:`, `chrome:`, `devtools:`, `view-source:` and extension URLs is blocked.

### A4. Enforcement in code, not prompt

- **Origin allow-list per task:** the user's request plus approved redirects (e.g. an OAuth identity provider). A new
  registrable domain needs approval.
- **Action classes** ([sandbox.md](sandbox.md) §7): form submit, send, post and upload are `transmit`; with `financial`,
  `destructive` and `permission` they show a main-process native confirmation with the origin, the action and the data
  to be sent. Neither the model nor the renderer can dismiss it.
- **Post-login sensitive pages are `permission` class:** account security, password and 2FA settings, API-token and
  app-password pages, connected-apps pages, OAuth consent (`/oauth/authorize`-style flows), email forwarding rules and
  billing. Detection is a maintained URL-pattern list plus page-text heuristics; ambiguous pages default to `permission`.
- **Untrusted-content wrapping:** page text reaches the model inside delimited untrusted blocks and can never change the
  tool allow-list or origin list. An optional "reader" pass summarizes long pages without tool access.
- Partition permissions denied by default: camera, microphone, geolocation, notifications, clipboard-read, USB/HID/serial,
  fullscreen, pointer lock, and extension installs.
- A visible indicator and live action log whenever the agent drives; a Stop control and Esc stop it instantly.

### A5. Profiles and lifecycle

- `<data dir>/browser-profiles/<id>/`, `0700`, cookies encrypted (Electron cookie-encryption fuse on).
- Profiles are user-named ("Work", "Shopping"); a task picks one; "Forget this site" clears its storage.
- Anonymous research tasks use an **ephemeral** partition.
- Agent tools are write-denied on profile dirs and on browsers' `NativeMessagingHosts` directories.
- **Linux:** if Electron `safeStorage` reports the `basic_text` backend (a hardcoded key; see the
  [safeStorage docs](https://www.electronjs.org/docs/latest/api/safe-storage)), Lyra refuses to persist logged-in
  partitions, uses an ephemeral partition and asks the user to unlock or install a keyring. Use the async availability
  API (the sync `isEncryptionAvailable()` is deprecated).
- **Ubuntu 24.04+:** the .deb ships an AppArmor profile for the Electron binary; the browser tool is disabled if the
  Chromium sandbox is not active. `--no-sandbox` is never used.

## Part B: Computer use

### B1. Architecture

One JSON-RPC contract and one shared Rust policy crate (`lyra-cu-proto`: schema, allowed-app set, tier table, key
allow-list, rate limits, audit record, lease). Its generated JSON Schema is the source of truth; the Swift helper is
tested against the same fixtures.

| OS | Helper | Capture | Elements | Input |
|---|---|---|---|---|
| macOS | `LyraComputerUse.app` (Swift, `dev.thehenrylabs.lyra.computeruse`) | ScreenCaptureKit `SCContentFilter(includingApplications:)` | AX via AXorcist | `CGEventPostToPid` |
| Windows | `lyra-cu-windows.exe` (Rust, `windows` crate) | Windows.Graphics.Capture per allowed HWND | UI Automation patterns | `SendInput` with a PID check before every send |
| Linux | `lyra-cu-linux` (Rust: `ashpd`, `pipewire`, `reis`, `atspi`, `x11rb`) | Portal ScreenCast window grant; X11 only inside nested Xephyr | AT-SPI2 | libei via the RemoteDesktop portal (GNOME); off on KDE; XTest inside Xephyr only |

- **macOS TCC attribution.** macOS attributes permissions to the responsible process. The helper is launched through
  LaunchServices or registered as an `SMAppService` agent, so the grants land on the helper, not on Lyra or core.
- Core ↔ helper: Unix socket (macOS, Linux) or named pipe (Windows) with peer authentication plus the pairing nonce
  ([secrets.md](secrets.md) §6). The helper accepts commands only from Lyra's core.
- The helper is an actuator: every action has a deadline and a pause lease; losing the lease stops it.
- [trycua/cua](https://github.com/trycua/cua) `cua-driver-rs` (MIT) is a reference and conformance oracle only, never a
  runtime dependency (its default mode is not default-deny). Its AGPL-licensed perception module is excluded.

### B2. Tool surface (provider-neutral)

`screenshot()` · `ax_tree(app)` · `click(x, y | element)` · `type(text)` · `key(combo)` · `scroll` · `drag` ·
`open_app(id)` · `wait`.
- Coordinates are **pixels plus a display id**; adapters translate to each provider's native format ([providers.md](providers.md) §6).
- Screenshots include only allowed apps' windows. A whole-display capture is never taken.
- Accessibility-element actions are preferred over pixel clicks.
- **No script primitive** (`run_applescript`, `run_script`) by default. A scripted mode, if enabled, emits only mediated
  batches of the same actions through the same checks.
- Keys use a per-tier **allow-list** (letters, digits, arrows, Enter, Tab, Esc; modifiers only with allowed letters),
  not a block-list; system shortcuts (app switchers, launchers, Force Quit, screenshot keys, Super/Win combos) never pass.

### B3. Confinement

- **Allowed apps default to empty.** The user grants apps per task or permanently in settings.
- **Tiers per app category:** browsers `read` (screenshots only; web tasks go through Part A), terminals and IDEs
  `click` (no typing), others `full`. Users can lower tiers; raising one needs confirmation.
- The tier is checked **at event time against the target PID of the window under the event**, not just the frontmost
  app (fixes the time-of-check/time-of-use race, threat L-22).
- Always off-limits: password fields and secure input, keychain and credential prompts, OS privacy settings, polkit
  agents and `pinentry`, elevated windows and UAC, and Lyra itself.
- **Takeover detection:** real user input pauses the agent immediately; Esc is a global kill switch; an overlay shows
  that the agent is in control, with a Stop button. Rate limits apply; every action is logged with a thumbnail.
- `transmit`, `financial` and `destructive` classes apply (clicking "Send" asks for confirmation).
- Windows has no OS consent for UIA or `SendInput`: Lyra shows its own native consent card per app (name, signer, path,
  PID), persisted by signer + path hash. The capture border stays on; it is the consent signal.

### B4. Per-OS parity

Strength: **S** OS-enforced · **M** enforced by Lyra with a known race · **W** best effort.

| Control | macOS | Windows | Linux X11 | Linux Wayland |
|---|---|---|---|---|
| Allowed-app-only capture | **S** | **S** per window (border shown) | **W** on host → nested Xephyr only | **S** per user grant, app cannot choose; restore tokens stored in the vault |
| Input bound to target | **M** (sibling windows of one process not separated) | **M** (UIA exact; `SendInput` hits focus, so pre-send PID check) | Xephyr only | GNOME **M** (foreground lease: activate, verify, send, verify); KDE raw input off; wlroots/niri capture + AT-SPI only |
| OS consent | TCC on the helper **S** | none for unpackaged apps **W** → Lyra consent card (MSIX identity evaluated, spike S-VAULT-W5) | none **W** | portal dialogs **M** |
| Elevation | n/a | UIPI blocks elevated targets **S**; failure is silent, so postconditions are verified | polkit reachable on X11: denied by policy | compositor-isolated |
| Takeover / kill | event tap **M** | low-level hooks; injected flag separates Lyra's events **S-M** (spike S-Win-3) | XInput2 counters **M** | no global listening **W** → focus/pointer polling, overlay Stop, `GlobalShortcuts` portal where present, short leases |

Linux tiers shown in the UI: Tier 1 X11 apps in nested Xephyr; Tier 2 GNOME Wayland; Tier 3 KDE Wayland (capture and
AT-SPI only); Tier 4 other compositors (capture and AT-SPI only, or unsupported). For "operate any app" work, the honest
answer on X11 and for raw input on KDE/wlroots is the **VM desktop mode** (T2).

### B5. Dev builds

macOS TCC grants are keyed to the code signature. Ad-hoc signatures change every build, so **TCC grants reset per
build**; `xtask tcc-reset` clears stale entries. Peer authentication uses the pairing nonce only ([secrets.md](secrets.md) §6).

## Acceptance

| Test | Pass condition |
|---|---|
| BC-1 | An injected page cannot cause a submit without a confirmation (L-07) |
| BC-2 | The tool list has no cookie, storage, JS-eval or raw-CDP tool; a cookie read attempt fails (L-08) |
| BC-3 | Navigating to an API-token creation page produces a `permission` confirmation (L-41) |
| BC-4 | A fresh install has zero TCC grants and zero allowed apps (L-09) |
| BC-5 | 200 scripted actions produce no wrong-target event (L-22; spikes S5, S-Win-2) |
| BC-6 | TCC lists the helper, not Lyra or core; screenshots exclude non-allowed windows (L-40) |
| BC-7 | No script primitive in the tool list (L-23) |
| BC-8 | Login handoff works on GitHub, a Google account, Atlassian Cloud and one 2FA site (spike S3) |

Spikes: S3, S3a–S3d, S5, S-Win-1..3, S-Lin-1..4, S-X, S7-cu ([spikes.md](spikes.md)).
