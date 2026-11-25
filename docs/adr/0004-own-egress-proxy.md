---
status: accepted
date: 2026-11-26
decision-makers: maintainer
---

# Own egress proxy `lyra-egress` with a name-constrained, memory-only CA

## Context and problem statement

Sandboxed code reaches the network only through a proxy that enforces domain, method and path rules, blocks SSRF
after DNS resolution, and terminates TLS only where it must. The proxy must be the same binary on all three OSes and,
on Windows, run as an AppContainer peer for MXC's strict mode. Adopt or build? See [sandbox.md](../design/sandbox.md) §5.

## Decision drivers

* R3 and R4: fetch-only rules, SSRF guard with IP pinning, DNS only via the proxy, logged decisions.
* A per-session CA that is **name-constrained, held in memory and destroyed at exit** (L-37).
* The proxy must not become a confused deputy: no credential injection in v1.
* One binary on three OSes; a process boundary for crash and memory isolation.

## Considered options

* The srt proxy (Node).
* The Codex `network-proxy` crate (Rust, on a pre-1.0 `rama`).
* iron-proxy or smokescreen (Go sidecars).
* A Rust MITM library (hudsucker) or framework (rama, pingora).
* Build `lyra-egress` on hyper, rustls, rcgen and hickory-resolver.

## Decision outcome

Chosen option: **build `lyra-egress`** (about 4–5k LOC), one process per session, outside any sandbox. MITM only for
hosts whose rules carry a method or path limit; everything else is CONNECT with an SNI == Host check; opaque tunnels
are refused. The CA uses `rcgen` `NameConstraints.permitted_subtrees` limited to those hosts. The resolved-address
guard of srt and the domain-matcher rules of Codex are reused as specifications.

### Consequences

* Good, because neither srt nor the Codex proxy issues a name-constrained, memory-only CA, and both are open source,
  so their designs can be reused without their gaps.
* Good, because one binary gives identical behaviour on all three OSes.
* Bad, because hand-written CONNECT and HTTP edge cases are Lyra's to maintain.
* Bad, because tools that ignore `SSL_CERT_FILE`-style variables, and Schannel clients on Windows, fail closed on
  MITM'd hosts. This is documented rather than silently allowed.
* Bad, because `git push` and `npm publish` cannot run in the sandbox; they become core-performed `transmit`
  actions after a native approval.

### Confirmation

Spike S-E1 (RSS ≤ 150 MB, ≤ 1.5× wall time, CA accepted by Node, Python, Git-OpenSSL and Go; an off-list name never
validates). Tests SB-4, SB-8 and L-17 probes.

## Pros and cons of the options

### srt proxy

* Good, because its `filterRequest` and resolved-address guard are sound.
* Bad, because its CA is not name-constrained, its key sits in a temp dir, it terminates all CONNECTs, and it puts Node in the path.

### Codex network-proxy

* Bad, because it is unpublished, depends on an alpha framework, persists its CA certificate and has no name constraints.

### Go sidecars

* Bad, because they add a third runtime; smokescreen has no MITM, so no method or path rules.

### Rust libraries and frameworks

* Bad, because hudsucker has one maintainer, rama churns before 1.0, and pingora is reverse-proxy shaped.

## More information

Fallback if S-E1 fails: the srt proxy as a child process in decider mode, patched for the CA. Placeholder-token
credential injection limited to per-method/path rules may be added later as its own decision.
