---
status: accepted
date: 2026-11-26
decision-makers: maintainer
---

# Rust `lyra-core` as a separate signed binary over inherited stdio

## Context and problem statement

Lyra's core holds vault access, the DB key, the policy engine and approval verification. It needs its own OS code
identity (its own macOS keychain access group), no JIT, and direct use of the security code (sandbox, egress, vault,
peer authentication, policy), which is Rust on every OS anyway. Which language and process model should the core use?
See [architecture.md](../design/architecture.md) §2–§3.

## Decision drivers

* The process that holds secrets must have its own code identity and keychain group (L-01, L-13).
* No JIT in the trust-critical process, so Windows mitigation policies (ACG, CFG) can apply.
* Windows, macOS and Linux in v1 with one implementation of the security code.
* The same binary must run headless for the CLI and a later server mode.
* Solo-maintainer load and build-loop speed.

## Considered options

* A. TypeScript core in an Electron `utilityProcess`.
* B. Rust core binary launched by Electron main, JSON-RPC over inherited stdio.
* C. Hybrid: TypeScript orchestration with Rust sandbox, vault and policy behind N-API.
* D. Go core.

## Decision outcome

Chosen option: **B**. `lyra-core` is a separate signed Rust binary (macOS `LyraCore.app`,
`dev.thehenrylabs.lyra.core`; Windows `lyra-core.exe` in the MSIX; Linux root-owned `/opt/Lyra`). Electron main spawns
it by absolute path with an allowlisted environment and talks JSON-RPC over inherited stdio. The first frame is
`hello{protocolVersion, buildId, launchKey}`. TypeScript remains in the renderer, in Electron main and in the generated
protocol types. This follows the open-source [`openai/codex`](https://github.com/openai/codex) app-server pattern.

### Consequences

* Good, because the core gets its own identity and keychain group, which a `utilityProcess` (sharing the Electron
  Helper identity and its `allow-jit`) cannot.
* Good, because the security crates link directly; no N-API layer for six targets.
* Good, because local probes measured under 10 ms start and about 10 MB RSS for a Rust core with the real dependency
  set, against about 150 ms and 173 MB for Node.
* Bad, because no vendor ships an official Rust SDK, so Lyra writes provider adapters (ADR 0012).
* Bad, because the maintainer carries TypeScript, Rust and Swift (Swift only for the macOS computer-use helper).
* Bad, because Windows libuv stdio pipes have a default DACL; main aborts on any spawn or pipe error, with a native
  `PROC_THREAD_ATTRIBUTE_HANDLE_LIST` spawn planned.

### Confirmation

Spikes S-CORE3 (build loop: ≤ 10 s incremental check, ≤ 15 min clean CI release), S-CORE4 (relay latency),
S-CORE5 (Windows mitigations) and S-CORE2 (identity; needs production signing for macOS).

## Pros and cons of the options

### A. TypeScript core

* Good, because it is the least effort and uses official TS SDKs.
* Bad, because core keeps the Electron Helper identity and JIT, and security code sits behind N-API.

### C. Hybrid

* Bad, because policy evaluation and approval verification stay in a JIT Node process; if the Rust parts get their own
  processes and authority, it converges on B with an extra hop.

### D. Go core

* Good, because official Go SDKs exist for the major providers.
* Bad, because there is no maintained SQLCipher binding or open Windows/Linux sandbox in Go, so Lyra would carry three
  native languages.

## More information

Revisit if S-CORE3 fails after crate splitting and sccache, or if S-CORE1 forces the TS provider worker (ADR 0012).
