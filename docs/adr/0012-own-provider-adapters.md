---
status: accepted
date: 2026-11-26
decision-makers: maintainer
---

# Own Rust adapters for four model wire formats

## Context and problem statement

Lyra is provider-neutral: it must talk to vendor APIs, gateways and self-hosted servers directly from the user's
machine. With a Rust core (ADR 0001) there is no official vendor Rust SDK, and Lyra must own the HTTP body anyway
(field allow-list, host pinning, no cross-host redirects). How does core talk to models?
See [providers.md](../design/providers.md).

## Decision drivers

* P2 and P4: direct calls; only allow-listed fields leave; pinned hosts (L-04, L-18).
* Native features kept: reasoning replay, prompt caching, computer-use tools.
* Bounded maintenance for one maintainer.
* Secrets stay in core; no other process sees the vault.

## Considered options

* Own adapters for four wire formats (OpenAI Responses, OpenAI Chat-compatible, Anthropic Messages, Gemini `generateContent`).
* Community Rust crates as dependencies (`genai`, `async-openai`, Rig provider modules).
* A sandboxed TypeScript worker running the Vercel AI SDK providers.

## Decision outcome

Chosen option: **own adapters** on `reqwest` (rustls) with an own SSE parser, within a 7k LOC budget for all four
formats. The Chat-compatible adapter covers vLLM, SGLang, LiteLLM, xAI, DeepSeek and Vercel AI Gateway. `genai` is a
differential test oracle only. Recorded-fixture contract tests run per provider in CI.

Fallback, triggered by the S-CORE1 kill criterion: a sandboxed TypeScript provider worker (AI SDK providers) that core
drives per request. It receives per-request headers only and never has vault access; core keeps the loop and policy.

### Consequences

* Good, because Lyra controls exactly which bytes leave and where they go.
* Good, because new OpenAI-compatible servers need no new adapter.
* Bad, because adapters are Lyra's to maintain forever as vendor APIs change; mitigated by a monthly vendor-changelog
  pass, fixtures and the oracle.
* Bad, because the fallback would let a Node process see each request's auth header.

### Confirmation

Spike S-CORE1: more than 3 of 20 tasks failing from adapter defects on any target, or the four formats exceeding 7k LOC,
triggers the fallback. Spike S-AF6 compares reference adapters. Tests PR-1..PR-6.

## Pros and cons of the options

### Community crates

* Good, because they already implement several formats.
* Bad, because `genai` has essentially one maintainer, Rig is a framework that would own the loop and makes frequent
  breaking changes, and `async-openai` is OpenAI-only.

### TypeScript worker

* Good, because the AI SDK has the broadest first-party coverage.
* Bad, because it adds a Node process to the request path and a second runtime to sign and update on three OSes.

## More information

Revisit if an official Rust SDK appears for a major vendor, or if the adapter budget is exceeded.
