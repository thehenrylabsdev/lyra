---
status: accepted
date: 2026-11-26
decision-makers: maintainer
---

# Electron 45 as the desktop shell on all three OSes

## Context and problem statement

The renderer displays untrusted model output, so it needs a strong renderer sandbox, a strict CSP and no Node in the
page. The agent browser needs CDP **without a TCP debugging port**. Both must behave the same on Windows, macOS and
Linux. Which shell? See [architecture.md](../design/architecture.md) §10 and
[browser-and-computer-use.md](../design/browser-and-computer-use.md).

## Decision drivers

* Uniform renderer security: sandbox, context isolation, CSP with a per-load nonce, Trusted Types (L-35, L-10, L-11).
* An in-process, portless CDP for the agent browser on every OS (L-08).
* Fuse-level hardening and ASAR integrity.
* Maintenance carried by a funded upstream, with fast security releases.

## Considered options

* Electron 45.
* Tauri 2 (OS webviews: WebView2, WKWebView, WebKitGTK).
* Tauri 3 with the CEF runtime (alpha).
* An own Rust shell on CEF.
* Native toolkits (Flutter, Slint, GPUI) or Qt WebEngine.

## Decision outcome

Chosen option: **Electron 45.5.0** (Node 24.22.3, Chromium 156) on all three OSes, with Electron Forge 8.0.1 and the
full fuse set. The browser executor runs `playwright-core` over `webContents.debugger` in main.

### Consequences

* Good, because one Chromium means one security test suite: same fuses, CSP and in-process CDP everywhere.
* Good, because Electron ships same-day patch releases on several branches.
* Bad, because Electron lags Chrome stable by about three majors. Policy: ship every Electron security release within
  7 days; reconsider if the lag exceeds 4 majors for 2 consecutive releases.
* Bad, because ASAR integrity is not available on Linux. Compensation: root-owned `/opt/Lyra` install and a launcher
  check of a signed hash list before core starts.
* Bad, because Ubuntu 23.10+ restricts unprivileged user namespaces; the .deb/.rpm ships an AppArmor profile and
  `chrome-sandbox`; `--no-sandbox` is never used.
* Bad, because downloads are 115–151 MB per platform.

### Confirmation

CI reads the fuses of every artifact (H1) and asserts a sandboxed renderer on each OS (H11). Spikes S-D1, S-D2,
S-SH2, S-SH4, S3a.

## Pros and cons of the options

### Tauri 2

* Good, because it is smaller and has a capability ACL.
* Bad, because WKWebView and WebKitGTK have no CDP, so the agent browser needs a second engine on two OSes; engine
  patching follows the OS; build-time CSP nonces conflict with a per-load nonce.

### Tauri 3 + CEF

* Bad, because it is alpha, its docs state the Windows renderer currently runs unsandboxed, and macOS bundles need
  `disable-library-validation`.

### Own Rust + CEF shell

* Bad, because it replaces Electron's whole main-process surface (estimated 12–25k LOC) and adds a CEF upgrade every
  Chromium major.

### Native toolkits, Qt WebEngine

* Bad, because they have no embedded browser (native toolkits) or lag Chromium badly and add LGPL relinking duties (Qt).

## More information

Re-evaluate when Tauri's CEF runtime is stable with a working Windows sandbox and no `disable-library-validation`
(watch item S-SH6).
