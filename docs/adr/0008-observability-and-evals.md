---
status: accepted
date: 2026-11-26
decision-makers: maintainer
---

# OpenTelemetry GenAI spans, Langfuse as an optional export target, promptfoo and Inspect AI in CI

## Context and problem statement

Lyra needs traces of agent runs for users and developers, and an evaluation harness that tests the real policy engine,
sandbox and approval protocol. Telemetry is off by default and nothing leaves the machine without opt-in
([architecture.md](../design/architecture.md) §9). Which observability and eval tools fit?

## Decision drivers

* Zero egress by default (L-26, L-27); export only to a user-named endpoint added through the native dialog.
* No SDK that monkey-patches clients or sits in the model request path.
* Content capture off by default and always redacted.
* Evals must drive the real `lyra-core`, not a mock.

## Considered options

* OpenTelemetry GenAI semantic conventions with local spans and optional OTLP export.
* Vendor SDKs: Langfuse SDK, OpenLLMetry, Opik SDK, Helicone (proxy), Arize Phoenix, MLflow.
* Eval tools: promptfoo, Inspect AI, garak, PyRIT, DeepEval, Ragas.

## Decision outcome

Chosen options:
- **Local spans always:** a projection turns event-log entries into OTel spans following GenAI semconv v0.1.0
  (`invoke_agent`, `chat`, `execute_tool`, MCP spans), stored in `lyra.db` and shown in the UI trace viewer. Rust
  `tracing` + `tracing-opentelemetry` 0.34.0; the `opentelemetry-otlp` 0.33.1 exporter is compiled behind an
  off-by-default feature and loaded only on opt-in.
- **Export, opt-in:** OTLP/HTTP to a user-named endpoint (an "add endpoint" trusted decision, listed in the egress
  catalogue). **Langfuse** (MIT core) is the documented target; any OTLP collector works. Content capture is a separate
  per-endpoint switch and always passes through redaction.
- **CI:** promptfoo 0.126.0 and Inspect AI 0.3.292 drive headless `lyra-core` over stdio JSON-RPC through small
  bridges (about 150–300 LOC each); a scripted approver answers through the protocol. **Dev only:** garak, PyRIT.
  Eval tools live in a separate `evals/` lockfile with telemetry opt-outs and a CI egress allowlist.

### Consequences

* Good, because traces cost nothing when export is off, and the schema is vendor-neutral.
* Good, because evals exercise the real policy engine, sandbox and approval path.
* Bad, because GenAI semconv is still "Development" and v0.1.0 already renamed spans; only the projection module
  changes, and the schema version is pinned in exported resource attributes.
* Bad, because promptfoo is now owned by OpenAI; Inspect AI is the second harness and cases stay in Lyra's own format.

### Confirmation

Spike S-AF3 (no packet outside the catalogue on a fresh install; no `gen_ai.*.messages` while capture is off) and
S-AF4 (harness on three OS runners in ≤ 20 min; reruns agree on ≥ 90% of cases). A canary-secret test on export.

## Pros and cons of the options

### Vendor SDKs

* Bad, because OpenLLMetry monkey-patches SDKs (247 packages), the Opik SDK renders templates unsandboxed, Helicone is a
  request-path proxy, and Phoenix is ELv2 (dev use only).

### DeepEval, Ragas

* Bad, because they overlap with promptfoo and Inspect; Ragas waits until retrieval exists (S-AF5).
