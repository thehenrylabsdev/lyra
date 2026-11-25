# Provider layer

Status: accepted design for v1 (ADR 0012). Gated by spike S-CORE1 ([spikes.md](spikes.md)).

## 1. Requirements

| Id | Requirement |
|---|---|
| P1 | Any model through: direct vendor APIs (OpenAI, Anthropic, Google Gemini, xAI, DeepSeek), LiteLLM, Vercel AI Gateway, or self-hosted vLLM / SGLang (OpenAI-compatible) |
| P2 | Lyra calls the user's endpoint **directly from the user's machine**. Keys and base URLs are never relayed through any Lyra or vendor backend |
| P3 | Features are detected per model, not per vendor name: tools, parallel tools, vision, reasoning, prompt caching, native computer use, structured output, context length |
| P4 | Lyra owns the HTTP body: only allow-listed fields leave, hosts are pinned at dial time, cross-host redirects are refused |
| P5 | Native vendor features are kept where the capability record says so (not lowest-common-denominator) |
| P6 | Model output is untrusted input to every tool (the prompt-injection boundary; [threat-model.md](threat-model.md)) |

## 2. Design

```
agent loop ──► ModelRouter ──► Endpoint (user-configured) ──► Adapter (one of 4 wire formats) ──► HTTPS (reqwest + rustls)
                  │                   │
                  │                   └─ capability record (declared + probed), host pin, secret references
                  └─ tier aliases: frontier · balanced · fast · local · judge
```

Lyra implements its own Rust adapters for four wire formats behind one internal model interface in `lyra-core`:

| Wire format | Used for |
|---|---|
| OpenAI Responses | OpenAI direct |
| OpenAI Chat Completions-compatible | vLLM, SGLang, LiteLLM, xAI, DeepSeek, Vercel AI Gateway, any OpenAI-compatible server |
| Anthropic Messages | Claude API (and Anthropic-compatible endpoints) |
| Gemini `generateContent` | Gemini API |

There is no official Rust SDK from these vendors. Each adapter is built on `reqwest` with Lyra's own SSE parser. The
community crate [`genai`](https://github.com/jeremychone/rust-genai) is used only as a differential test oracle, never
as a dependency. Budget: the four formats together MUST stay within 7k LOC (S-CORE1 kill criterion).

**Fallback (if S-CORE1 kills the own adapters):** a sandboxed TypeScript provider worker using the Vercel AI SDK
providers, driven by core per request. It receives per-request headers only and never has vault access. Core still
owns the loop, the policy engine and the event log.

Each adapter handles: streaming deltas, tool-call deltas (including parallel calls), tool errors, reasoning/thinking
blocks and their replay across tool turns, prompt-cache controls, abort mid-stream, and usage accounting.

## 3. Endpoint record

Stored in `lyra.db`. Secrets appear only as references.

```jsonc
{
  "id": "home-vllm",
  "kind": "openai-compatible",       // openai | anthropic | google | xai | deepseek | vercel-gateway | litellm | openai-compatible
  "baseURL": "https://gpu.example.lan/v1",
  "auth": { "type": "bearer", "secret": "${secret:home-vllm-key}" },  // none | bearer | header | oauth | command (short-lived)
  "headers": { "X-Org": "${secret:org-id}" },
  "pinHosts": ["gpu.example.lan"],   // dial-site allowlist; redirects to other hosts are refused
  "tls": { "caBundle": null, "clientCert": "${secret:mtls-cert}", "pinnedFingerprint": null },
  "models": "discover",              // or an explicit list
  "compat": { "allowFields": ["reasoning_effort", "chat_template_kwargs", "top_k", "min_p"] },
  "spendGuard": { "monthlyTokens": null, "hardStop": true }
}
```

Rules:
- Endpoint records are **user-layer only**. Project or workspace files cannot add or redirect an endpoint (threat L-04).
- Adding an endpoint is a trusted decision (native dialog, [architecture.md](architecture.md) §4).
- HTTPS is required except for explicitly allowed loopback or LAN hosts. TLS verification is never disabled silently;
  "trust this self-signed certificate" pins its SHA-256 fingerprint.
- Fields outside the wire format's schema are dropped unless listed in `compat.allowFields`.

## 4. Capability record and tier aliases

- **Capability record per model** = declared (adapter defaults or the gateway's model list) + **probed**. The probe is
  opt-in because it costs tokens. It sends one tool call with a schema that has nested objects and a long string
  argument (simple schemas hide known server-side parser faults), one vision input, and one reasoning turn.
- The router refuses to send tools to a model without tool support, then falls back to another model of the same tier
  or tells the user.
- **Tier aliases** (`frontier`, `balanced`, `fast`, `local`, `judge`) map to concrete models per user. Subagents,
  compaction, titles and the approval judge each use a tier, so swapping providers never edits code.
- The **judge** tier can be a local model, so advisory risk annotation works behind any gateway. A judge never approves
  `transmit`, `destructive`, `financial` or `permission` actions ([sandbox.md](sandbox.md) §7).

## 5. Gateways and self-hosted servers

### LiteLLM
- An untrusted backend the user points Lyra at. Its 2026 advisories include pre-auth SQL injection
  ([CVE-2026-42208](https://www.cve.org/CVERecord?id=CVE-2026-42208)) and MCP test-endpoint command execution
  ([CVE-2026-42271](https://www.cve.org/CVERecord?id=CVE-2026-42271)); new advisories appear often, so only the latest
  release is a safe test target.
- Lyra does **not** bundle LiteLLM. If a later version does: bind to a Unix socket or `127.0.0.1`, generate the master key
  into the vault, disable the admin UI, config-write endpoints and its MCP gateway, and gate on the newest fixed version.
- Lyra MAY detect LiteLLM from response headers to label "via LiteLLM" in the UI; trust does not change.

### Vercel AI Gateway
- `https://ai-gateway.vercel.sh/v1`; auth by an AI Gateway key in the vault, or OIDC. BYOK is configured on the gateway side.
- Compatibility endpoints may not expose every provider-native field, so capability probing applies.

### vLLM and SGLang
- OpenAI-compatible `/v1/chat/completions`. Tool calling depends on server flags (tool-call parser, reasoning parser),
  so it MUST be probed. Typical deployments are LAN or tailnet; custom CA and mTLS are supported.
- Reasoning arrives as `reasoning_content` (or `reasoning` on newer servers) in messages and deltas; the adapter reads both.
- Non-OpenAI request fields (`chat_template_kwargs`, `top_k`, `min_p`) need `compat.allowFields`.
- The adapter MUST treat "the model said it would call a tool but no tool call was parsed" as a recoverable error,
  and MUST validate or reject tool arguments that arrive as JSON strings instead of objects.

Reference test target (ADR 0011): `Qwen/Qwen3.8-27B` (Apache-2.0) on vLLM 0.34.0 with
`--enable-auto-tool-choice --tool-call-parser qwen3_coder --reasoning-parser qwen3`, served on `127.0.0.1` with
`--api-key` and reached through an SSH tunnel, so Lyra's endpoint is `http://127.0.0.1:8000/v1` (the pinned-loopback path).
SGLang 0.5.23 is the second server.

## 6. Native vendor features

Adapters pass through, when the capability record allows: prompt caching (Anthropic, Gemini), reasoning effort,
extended thinking and its replay, native computer-use tools, web-search tools and file inputs.

Computer-use tool formats (pixel coordinates plus a display id inside Lyra; see
[browser-and-computer-use.md](browser-and-computer-use.md)):

| Provider | Format |
|---|---|
| Anthropic (Claude API) | `computer_toolset_20260801`. The older `computer_20251124` tool is rejected by current models (Opus 5.5, Sonnet 5.5, Fable 5.2) on the Claude API, per the Claude API reference |
| OpenAI | `{"type": "computer"}` |
| Gemini | `computer_use` |
| Others | Plain function tools |

## 7. Security rules

1. Endpoint config is user-layer only (L-04).
2. Host pinning at dial time; no cross-host redirects; HTTPS except allowed loopback/LAN.
3. Secrets are resolved from the vault per request and never logged. Request logs strip `Authorization`, `x-api-key`,
   `api-key` and `cookie`. Debug body dumps are redacted and prove that no non-allow-listed field left.
4. Optional per-endpoint spend guard: a monthly token cap with a hard stop.
5. Network capture in tests shows keys only to the configured host (L-18).
6. `auth.type = "command"` (a short-lived token helper) runs sandboxed; its output is treated as a secret.

## 8. Acceptance

| Test | Pass condition |
|---|---|
| PR-1 | The same conversation runs on three different endpoint kinds; no key appears in logs or the DB |
| PR-2 | A redirect from a pinned host to another host is refused |
| PR-3 | A request-body dump contains only schema fields and `compat.allowFields` |
| PR-4 | A project `.lyra/config.json` that sets a `baseURL` is ignored with a warning (also ST-5) |
| PR-5 | Abort mid-stream stops server-side generation within 5 s (spike S6-tunnel) |
| PR-6 | Recorded-fixture contract tests per provider pass in CI with no network |

Spikes: S-CORE1, S-AF6, S1-vLLM, S1b-parsers, S1c-reasoning, S6-tunnel, S7-cu ([spikes.md](spikes.md)).
