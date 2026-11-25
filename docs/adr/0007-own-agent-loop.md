---
status: accepted
date: 2026-11-26
decision-makers: maintainer
---

# Own agent loop and durable turn state machine; no agent framework

## Context and problem statement

Every tool call in Lyra is a trusted decision: the deterministic policy engine classifies it, gated calls need a
main-minted token from a native dialog, the call runs over the actuator protocol with a deadline, idempotency key and
lease, and it is recorded in the append-only event log. Should an agent framework or a durable-execution engine run
the loop? See [architecture.md](../design/architecture.md) §5–§6.

## Decision drivers

* No framework approval state may count as an approval; frameworks may only pause (L-35, L-06).
* Tool execution never happens inside a framework; all executors are sandboxed (L-03).
* One state store: the encrypted event log.
* Crash safety: gated actions are never executed twice.
* A Rust core (ADR 0001).

## Considered options (evaluated)

| Option | Verdict |
|---|---|
| LangGraph (JS, Python) | Reject: a second state store beside the event log, at least 8 CVEs in its checkpoint and serializer layers, no Rust version. Its `interrupt` semantics are borrowed |
| LangChain, LlamaIndex (Python), CrewAI, Google ADK, Microsoft Agent Framework | Reject: Python or heavy JS, large CVE surface, they want to execute tools |
| LlamaIndex.TS | Reject: archived 2026-03-11 |
| Mastra | Reject: open-core |
| OpenAI Agents SDK, Pydantic AI | Reject as dependencies; borrow `RunState` and deferred-tool designs |
| Rig, `genai` (Rust) | Reference and test oracle only |
| Temporal, DBOS | Reject for the desktop (need a server or Postgres); revisit for a server tier |
| Restate, Inngest | Reject: BSL and SSPL servers |
| Own loop + durable state machine | **Adopt** |

## Decision outcome

Chosen option: **own loop** (about 2–3k LOC Rust) **plus a durable turn state machine** (about 1–1.5k LOC) over the
SQLite event log. Lifecycle per tool call: `proposed → policy_evaluated → approval_requested → approval_decided →
dispatched → completed | failed | lease_lost | interrupted`. On restart, a `dispatched` call is resolved by asking the
executor about its idempotency key; `transmit`, `destructive`, `financial` and `permission` calls are **never retried
automatically**.

### Consequences

* Good, because the approval protocol, item model and leases are the product, and they stay in one auditable place.
* Good, because the loop is OS-neutral; only lease-loss enforcement uses per-OS primitives (job objects, process groups, cgroups).
* Bad, because memory, loaders and multi-agent patterns are built when needed rather than inherited.
* Bad, because recovery bugs are Lyra's; mitigated by idempotency keys, property tests and the S-AF1 crash matrix.

### Confirmation

Spike S-AF1: kill -9 at 7 lifecycle points × 3 action classes on three OSes, with zero duplicate gated actions, zero
lost approvals and zero writes after lease expiry.

## More information

Revisit for a multi-node server tier with long-running jobs (then Temporal or DBOS). The earlier plan for a
LangGraph.js bake-off is dropped because a Rust core would put the trust-critical loop back into Node.
