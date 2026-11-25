# Lyra

**One desktop app for every AI model and agent, with security you can verify.**

Lyra brings chat, a coding agent, a browser agent and computer use together in one open-source app for Windows,
macOS and Linux. You choose the models: a frontier API, your company's gateway, or a model you host yourself. Lyra
treats every model the same, and treats every agent action as something you can see, limit and approve.

> **Status: pre-alpha, design stage.** Nothing is released yet. The architecture and security design are done. The
> UX/UI is being designed next, and the core follows later. See the [roadmap](docs/plan/ROADMAP.md).

---

## Why Lyra

Today's AI tools make you choose. Each one is tied to a vendor's models, keeps your API keys wherever it likes, and
runs agents with more access than they need. Lyra is built on three commitments.

**1. Your models, not ours.**
Connect any of these and switch per conversation, or route by task:
- OpenAI, Anthropic, Google, xAI or DeepSeek;
- any OpenAI-compatible server (vLLM, SGLang, Ollama-style endpoints);
- a gateway such as LiteLLM or the Vercel AI Gateway.

Lyra makes no calls to a vendor of its own and sends no telemetry unless you turn it on.

**2. Agents that can't overreach.**
- **Coding agent:** commands run in a default-deny sandbox on every OS. It can read and write only the paths a task
  needs, and it reaches the network only through an audited proxy that you control.
- **Browser agent:** works in a separate browser profile you log into yourself. The model never sees your passwords
  or cookies.
- **Computer use:** sees and controls only the apps you allow, and a visible Stop control is always on screen.
- **Approvals:** anything that sends, deletes, pays or changes permissions needs your approval in a native dialog,
  which a compromised UI can't forge.

**3. Secrets that stay secret.**
API keys and tokens live in an encrypted vault backed by your OS (Keychain, TPM, Secret Service). They never enter
the model's context, the sandbox, logs or config files. Lyra has no plaintext fallback. Where an OS can protect less,
for example because Windows and Linux key stores are per-user, Lyra tells you plainly and tells you how to raise your
protection.

## What you'll be able to do

| | |
|---|---|
| **Chat** | Any model, side by side; streaming, attachments, search across all your threads |
| **Code** | An agent that plans, edits, runs tests and shows diffs, with checkpoints you can roll back. `git push` happens only after you approve |
| **Browse** | Hand a task to an agent in a browser you're logged into; confirm before anything is submitted |
| **Operate apps** | Computer use, limited to the apps you choose |
| **Connect tools** | MCP connectors for GitHub, Atlassian, Slack, Figma, Google Drive, Telegram and Microsoft Teams, with scoped OAuth and tokens in the vault |
| **Use other agents** | Run Claude Code, Codex, Gemini CLI and others inside Lyra, through the Agent Client Protocol, with Lyra's approvals around them |

## How it works

```
React UI (untrusted)
   │ typed IPC
Electron main ── native approval dialogs, browser host
   │ JSON-RPC over an inherited pipe
lyra-core (Rust) ── agent loop · model adapters · MCP · encrypted event log · vault · policy engine
   ├── lyra-sandbox   Seatbelt (macOS) · bubblewrap + seccomp (Linux) · MXC containers (Windows)
   ├── lyra-egress    the only network path for sandboxed code: allowlists, SSRF guard, audit
   └── computer-use helpers, one per OS
```

- **Rust core.** The process that holds your secrets and makes policy decisions is small and memory-safe. It has its
  own signed identity on each OS.
- **Same rules on every OS.** One policy engine and one proxy, with an OS-native sandbox on each platform. Where an
  OS's protection differs, the design document says so and the app shows it.
- **Open design.** Read the [design docs](docs/design/README.md) and the [architecture decision records](docs/adr/).

## Platforms (v1 target)

| OS | Minimum |
|---|---|
| macOS | 15 |
| Windows | Windows 11 (24H2 or later, at the build the Windows sandbox requires) |
| Linux | Ubuntu 22.04+, Debian 12+, current Fedora (.deb/.rpm) |

## Roadmap

1. **Foundation:** repository, governance and design. *Done.*
2. **UX/UI:** design system and every flow as a clickable prototype. *Next.*
3. **Core:** Rust core, vault, model adapters, sandbox, egress proxy, coding agent, then connectors, the browser agent
   and computer use. *Planned.*

The full plan, broken into epics, stories and one-PR subtasks, is in [docs/plan/ROADMAP.md](docs/plan/ROADMAP.md).

## Repository layout

| Path | What |
|---|---|
| [docs/design/](docs/design/README.md) | The specification: architecture, secrets, sandbox, providers, browser and computer use, MCP, threat model, stack |
| [docs/adr/](docs/adr/) | Architecture decision records |
| [docs/plan/](docs/plan/ROADMAP.md) | Roadmap and work breakdown |
| [CLAUDE.md](CLAUDE.md) | Working context for contributors and AI coding agents: repo map, invariants, workflow |

## Contributing

Contributions are welcome. Lyra is early, so design feedback is as valuable as code. Start with
[CONTRIBUTING.md](CONTRIBUTING.md), and please report security issues privately as described in
[SECURITY.md](SECURITY.md).

## Licence

[Apache License 2.0](LICENSE). Third-party attributions are in [NOTICE](NOTICE).
