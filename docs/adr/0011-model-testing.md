---
status: accepted
date: 2026-11-26
decision-makers: maintainer
---

# Model testing uses the Claude API or a RunPod Qwen3.8-27B endpoint

## Context and problem statement

Provider adapters, the agent loop, prompt-injection defences and evals need real model calls. Frontier APIs cover
model quality; a self-hosted open model is needed for what only vLLM or SGLang exercise: server-side tool-call and
reasoning parsers, the `reasoning_content` field, and a weaker, injectable target for the guard layers. Which models
do tests and spikes use? See [providers.md](../design/providers.md) §5 and [spikes.md](../design/spikes.md) §7.

## Decision drivers

* Cover both a frontier wire format (Anthropic Messages) and the OpenAI-compatible self-hosted path.
* Keep cost bounded; every billable run is quoted to the maintainer first.
* Keep test endpoints private and test data synthetic.
* Most CI runs need no network: record once, replay from fixtures.

## Considered options

* Claude API for frontier tests.
* `Qwen/Qwen3.8-27B` (Apache-2.0, dense, vision-language) on vLLM 0.34.0 or SGLang 0.5.23, rented on RunPod.
* Other GPU clouds, or a local quantized model (validates the adapter, not vLLM's parsers).
* All vendor APIs for every spike.

## Decision outcome

Chosen option: **the Claude API or a RunPod `Qwen/Qwen3.8-27B` endpoint.** Other vendor endpoints are added to
S-CORE1 when credentials exist, but are not required.

Rules:
- Credentials come from the developer's environment or local SDK profile, never from the repository, CI logs or prompts.
- The Qwen endpoint runs vLLM 0.34.0 with `--enable-auto-tool-choice --tool-call-parser qwen3_coder --reasoning-parser qwen3`,
  bound to `127.0.0.1` with `--api-key`, reached through an SSH tunnel (no public inference port). The per-session API
  key goes into the vault.
- Use a Secure Cloud pod (not serverless), synthetic or public corpora only, no real secrets or private repos in
  prompts; delete the pod and volume at the end of each campaign and confirm nothing is left running.
- Record raw SSE once; CI replays fixtures at no GPU cost.
- Pin the model's Hugging Face commit SHA at download time; re-run the parser suite at each vLLM minor release.
- Attribute every failure to model, parser or adapter from raw logs before counting it against a kill criterion.

### Consequences

* Good, because two very different targets cover both wire-format families and the parser layer.
* Good, because the pinned-loopback endpoint also tests the provider layer's loopback path.
* Bad, because known open vLLM parser issues can masquerade as adapter defects; mitigated by raw logging (S1b-parsers).
* Bad, because a third-party GPU host holds prompts in memory; mitigated by synthetic data and deletion after each campaign.

### Confirmation

Spikes S-CORE1, S1-vLLM, S1b-parsers, S1c-reasoning, S6-tunnel and optionally S7-cu; each records its cost quote and
approval in its spike report.

## More information

If a Qwen3.8-27B parser blocker appears, fall back to an earlier 27B Qwen release on the same server.
