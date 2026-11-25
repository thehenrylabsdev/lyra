# Lyra design documentation

These documents are Lyra's working specification: requirements, interfaces, invariants and acceptance tests. They
are written for contributors and for coding agents that implement Lyra from them. Decisions and their rationale are
recorded as ADRs in [`docs/adr/`](../adr/). The delivery plan is in [`docs/plan/ROADMAP.md`](../plan/ROADMAP.md).

Lyra is a provider-neutral desktop AI app for Windows, macOS and Linux: chat across model providers, a sandboxed
coding agent, an agent-controlled browser the user logs into, computer use, and MCP integrations. Licence: Apache-2.0.

## Index

| Document | Covers |
|---|---|
| [architecture.md](architecture.md) | Process map, trust boundaries, native approval dialog and nonce, protocol (threads, turns, items, JSON-RPC, actuator), durable state machine, storage layout, config layering, egress catalogue, Electron hardening checklist |
| [secrets.md](secrets.md) | Envelope vault, assurance levels L0–L3, user guidance per OS, secret references, IPC peer authentication per OS, OAuth, acceptance tests |
| [sandbox.md](sandbox.md) | Tiers T0–T2, requirements R1–R10, per-OS parity matrix, egress proxy, preflight probes, permission engine |
| [providers.md](providers.md) | Four wire-format adapters, endpoint and capability records, tier aliases, gateways and self-hosted servers |
| [browser-and-computer-use.md](browser-and-computer-use.md) | Agent browser, login handoff, narrow tool API, computer-use helpers and per-OS confinement |
| [mcp-and-connectors.md](mcp-and-connectors.md) | MCP client requirements, connector manifests, the seven v1 integrations, ACP external engines |
| [threat-model.md](threat-model.md) | Adversaries, threat register L-01..L-42 with controls and tests, residual risks |
| [stack.md](stack.md) | Exact picks, licences, OS floors, per-OS parity summary, dependency rules |
| [spikes.md](spikes.md) | Every spike with its question, kill criterion, OS and duration, plus the id mapping |

## Reading order for implementers

1. **ADRs 0001–0012**, to know what is decided and why.
2. **architecture.md**: the shape every other document assumes. Read §3 (trust boundaries) and §4 (approvals) twice.
3. **threat-model.md**: what each control defends against; keep the `L-xx` ids in commit messages and tests.
4. **The document for your area:** secrets, sandbox, providers, browser and computer use, or MCP.
5. **spikes.md**: run the spikes that gate your area before building on them.
6. **stack.md**: pin exactly what it says; re-check versions under its dependency rules.

## Conventions

- MUST, MUST NOT, SHOULD and MAY follow RFC 2119.
- Ids are stable and never reused: threats `L-xx`; sandbox requirements `R1`–`R10`; tests `ST-*`, `A*`, `SB-*`,
  `BC-*`, `MT-*`, `PR-*`; hardening items `H*`; trust boundaries `TB-*`; spikes as in [spikes.md](spikes.md).
  A retired id stays listed as retired.
- Identifiers: app `dev.thehenrylabs.lyra`, core `dev.thehenrylabs.lyra.core`, macOS computer-use helper
  `dev.thehenrylabs.lyra.computeruse`, keychain access group `TEAMID.dev.thehenrylabs.lyra.vault`. Vault data is bound
  to these, so they must not change after the first signed release.
- Development builds are unsigned or ad-hoc signed until the production-signing epic (ADR 0010). Each document states
  how its controls behave in such builds.
- When a design changes, update the document in the same PR as the code, and add or supersede an ADR for a decision change.
