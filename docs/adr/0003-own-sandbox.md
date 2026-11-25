---
status: accepted
date: 2026-11-26
decision-makers: maintainer
---

# Own `lyra-sandbox`: Seatbelt on macOS, bwrap + seccomp + netns + Landlock on Linux, MXC on Windows

## Context and problem statement

Everything the agent executes (shell, file tools, MCP stdio servers, hooks, plugins) must run confined, with
deny-default reads and writes and network only through Lyra's proxy, at equal strength on three OSes. Should Lyra
adopt an existing sandbox or build its own launcher and policy compiler? See [sandbox.md](../design/sandbox.md).

## Decision drivers

* Requirements R1–R10, in particular deny-default **reads** (L-02), proxy-only network (L-37), no sockets (L-38).
* No silent downgrade on any OS; an explicit `sandbox_unsupported` state.
* Windows confinement without UAC, a sandbox user or ACL stamping.
* Exact Lyra policy semantics everywhere, tested by one probe suite.

## Considered options

* [`anthropic-experimental/sandbox-runtime`](https://github.com/anthropic-experimental/sandbox-runtime) (srt).
* Vendoring the [`openai/codex`](https://github.com/openai/codex) sandbox crates.
* `mxc-sdk` on all three OSes.
* Build `lyra-sandbox`, using MXC on Windows only.
* Windows Sandbox or WSL2 as the Windows backend.

## Decision outcome

Chosen option: **build `lyra-sandbox`** (Rust, about 12–13k LOC) with one backend per OS:
- **macOS:** Lyra-generated SBPL applied with `/usr/bin/sandbox-exec` (absolute path); `sandbox_init` FFI if the CLI is removed.
- **Linux:** bundled, non-setuid bubblewrap 0.13.1, a seccomp filter, `--unshare-net` with a Unix-socket bridge to the
  proxy, and Landlock as a second layer. The .deb/.rpm installs a narrow AppArmor profile for Lyra's bwrap.
- **Windows:** Microsoft MXC ProcessContainer (`base-container` tier only) via `mxc-sdk` =1.1.0, built with telemetry
  unrouted, plus an outer job object.

Codex's SBPL fragments and seccomp design and srt's mach-lookup allowlist and denied ranges are copied with
attribution (Apache-2.0).

### Consequences

* Good, because MXC is an OS-native, default-deny container that needs no elevation.
* Good, because the macOS and Linux code is small and well understood, so policy semantics hold exactly.
* Bad, because it is about 12–13k LOC to build and maintain; mitigated by golden-file tests of generated profiles
  and argv, and one cross-OS probe suite.
* Bad, because MXC is new (GA October 2026), has required telemetry (unrouted at build, ETW-checked in CI) and a build
  floor that excludes Windows 10 (ADR 0006).
* Bad, because macOS process limits and tree kill are weak; the proxy revokes the session token at kill, and T2 is
  the remedy for hard limits.

### Confirmation

Preflight probes (R9, test SB-1) on every release; spikes S-W1..S-W3, S-L1, S-L2, S-M1, S-M2 and S2.

## Pros and cons of the options

### srt

* Good, because it works on macOS and Linux and was a strong baseline.
* Bad, because its own documentation lists Windows network-fence bypasses (anonymous or self-derived tokens, unfenced
  DNS) and broken Schannel TLS, its read model is allow-by-default, and its MITM CA is not name-constrained.

### Codex crates

* Bad, because they are not published on crates.io and are tied to internal protocol types; copying files is cheaper.

### MXC everywhere

* Bad, because on macOS and Linux it adds about 300k lines for what Lyra does in about 5k.

### Windows Sandbox, WSL2

* Bad, because `wsb exec` has no process I/O and needs Pro editions; WSL2 automounts the Windows drive with the user's rights.

## More information

Revisit Windows if MXC regresses (then an own AppContainer + LPAC backend); Linux if S-L1 fails (Landlock-only mode at ABI ≥ 6, disclosed as M).
