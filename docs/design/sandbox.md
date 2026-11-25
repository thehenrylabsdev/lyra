# Sandbox, egress proxy and permission engine

Status: accepted design for v1 (ADR 0003, ADR 0004).

**Principle.** The OS sandbox is the security boundary. Permission rules and any LLM judge are UX on top of it.
Everything the agent causes to execute runs confined, and there is no model-settable escape.

## 1. Execution tiers

| Tier | Mechanism | Default for | Isolation |
|---|---|---|---|
| **T1 host sandbox** (default) | `lyra-sandbox` (Rust). macOS: generated Seatbelt (SBPL) profile applied with `/usr/bin/sandbox-exec` (absolute path). Linux: bundled bubblewrap + seccomp + network namespace bridged to the proxy over a Unix socket, Landlock as a second layer. Windows: Microsoft MXC ProcessContainer (`base-container` tier) via `mxc-sdk`, plus an outer job object | Interactive coding, file tools, MCP stdio servers, hooks, plugins | Deny-default read and write; network only via `lyra-egress` |
| **T2 VM** (after spike S-VM) | libkrun via microsandbox on all three OSes, virtio-net into a user-space netstack (never TSI). The guest works on a **copy or git worktree**, never the live tree. Changes sync back through the T1 mandatory write-deny filter plus a diff review | Unattended or background runs, untrusted repos, container builds | Separate kernel; host credentials absent |
| **T0 unsandboxed** | Never offered to the model. Only the user can run one command at a time, from a main-process native dialog showing the exact command | Rare escape (a tool needing a system service) | None, hence human-only |

Fallbacks if S-VM fails: Apple `containerization` on macOS 26+, libkrun directly on Linux; Windows then has **no T2**
and unattended runs are disabled there. If Apple removes `sandbox-exec`, the macOS backend switches to calling
`sandbox_init` between `fork` and `exec` (the approach `microsoft/mxc` uses).

## 2. Requirements (stable ids)

| Id | Requirement |
|---|---|
| R1 | Deny-default **read and write**. Policy roots: workspace, toolchains (read-only), tmp, caches. One profile per command, inherited by its children |
| R2 | Mandatory, non-overridable denies. Read-deny: the credential paths in [secrets.md](secrets.md) §5. Write-deny: §3 below |
| R3 | Network only via `lyra-egress`. Rules are domain + method + path. Private, loopback, link-local and metadata IPs are blocked after DNS with the dialled IP pinned. Fetch-only by default. DNS is resolved by the proxy only. Every decision is logged |
| R4 | Selective TLS termination with a per-session CA that is **name-constrained, held in memory and destroyed at exit** |
| R5 | No Unix sockets or named pipes: `docker.sock`, `SSH_AUTH_SOCK`, `\\.\pipe\docker_engine`, `\\.\pipe\openssh-ssh-agent`; no Apple Events |
| R6 | Process limits and a wall-clock watchdog; clean kill of the whole process tree |
| R7 | No inherited secrets: the environment is built from an allowlist; an MCP server gets only its mapped secrets |
| R8 | Shell, file tools, MCP stdio servers, hooks and plugins all run under T1 |
| R9 | Preflight probe suite per OS version. Refuse T1 if any probe succeeds; report `sandbox_unsupported` explicitly |
| R10 | T2 VM: separate kernel, works on a worktree copy, egress only via `lyra-egress` |

Windows, macOS and Linux ship at equal strength where the OS allows; every gap is documented with its compensating
control, and nothing is silently downgraded.

## 3. Policy

Generated from a typed `SandboxPolicy`:

```jsonc
{
  "fs": {
    "read":  ["${workspace}", "${toolchains}", "${system}", "${tmp}"],
    "write": ["${workspace}", "${tmp}", "${caches}"],          // caches: per-workspace npm/pip/cargo dirs
    "denyRead":  "${MANDATORY_READ_DENY}",                     // not overridable
    "denyWrite": "${MANDATORY_WRITE_DENY}"                     // not overridable
  },
  "network": { "mode": "proxy",
               "allow": [ { "host": "registry.npmjs.org", "methods": ["GET", "HEAD"] },
                          { "host": "pypi.org", "methods": ["GET", "HEAD"] },
                          { "host": "files.pythonhosted.org", "methods": ["GET", "HEAD"] },
                          { "host": "github.com", "methods": ["GET", "HEAD", "POST"],
                            "paths": ["*/info/refs", "*/git-upload-pack"] } ],
               "deny": [], "privateRanges": "deny" },
  "process": { "allowExec": true, "wallClockSeconds": 1800 },
  "ipc": { "machLookup": "minimal", "appleEvents": false, "unixSockets": "deny" }
}
```

- `${toolchains}` resolves the user's language runtimes (e.g. `~/.nvm/versions/node/*`, `~/.cargo`, `~/.pyenv`), read-only.
- **Fetch-only by default.** Registry publish endpoints and `git-receive-pack` (push) are unreachable. Push and publish
  are `transmit` actions that core performs after approval, which closes the route where injected code pushes the
  workspace to an attacker's repository with the attacker's own token.
- **Mandatory write-deny:** `.git/hooks/**`, `.git/config`, the directory named by `core.hooksPath` (resolved at build
  time), `.husky/**`, `.pre-commit-config.yaml`, `.lyra/**`, `.vscode/**` tasks and settings, `.idea/**` run
  configurations, other agent tools' project config (a maintained list, e.g. `.claude/settings*`, `.mcp.json`), `.envrc`,
  shell rc files, autostart paths (`~/Library/LaunchAgents`, `~/.config/autostart`, the Windows Startup folder and Run
  keys), browser `NativeMessagingHosts` directories, Lyra's data dir, and any `node_modules/.bin` outside the workspace.
- Paths are canonicalized and symlinks resolved when the policy is built. A volume root is refused as a write root.
- **Never reachable:** Docker, Podman, OrbStack and Colima sockets, `SSH_AUTH_SOCK`, `launchd` user agents. Docker
  access is a full escape, so container builds need T2 or a human-run T0 command.

## 4. Per-OS parity matrix (T1)

Strength: **S** kernel- or OS-enforced, no known bypass · **M** enforced with known caveats · **W** weak.

| Req | macOS (Seatbelt) | Linux (bwrap + seccomp + netns + Landlock) | Windows (MXC BaseContainer) | Gap → compensating control |
|---|---|---|---|---|
| R1 read | `(deny default)` with `file-read*` subpath allows. **S** | Only allowed roots bind-mounted read-only; Landlock read rules. **S** | `readonlyPaths` default-deny; baseline `ALL_APPLICATION_PACKAGES` ACEs still expose system dirs. **M** | Windows: credential paths explicitly in `deniedPaths`; probes read `%USERPROFILE%\.ssh` and `%APPDATA%\gh` |
| R1 write | `file-write*` only on roots. **S** | `--bind` only on roots; Landlock write rules. **S** | `readwritePaths`. **S/M** | Refuse a volume root |
| R2 | Deny rules after allows, literal plus resolved paths. **S** | `--ro-bind` over each path; `/dev/null` over missing paths or symlink components. bwrap carries R2 (Landlock cannot deny below an allowed dir). **S** | Write-deny via `readonlyPaths` beneath a rw root (more specific wins); read-deny via `deniedPaths`; hard-link names keep their own rights. **M** | Windows: refuse a workspace where a protected file has link count > 1; create missing protected dirs before launch |
| R3 network | Only `localhost:<egress port>`; `mDNSResponder` and `trustd` denied, so no DNS. **S** | `--unshare-net` (only `lo`); TCP → Unix-socket bridge to `lyra-egress`; no DNS route. **S** | Per-container WFP allows only the proxy. **S** with `allowedProxyPeer` (proxy in its own AppContainer profile, spike S-W2); the identity-less mode is dev/test only and **release builds refuse it** | — |
| R3 rules, SSRF, DNS | Same `lyra-egress`. **S** | **S** | Same, but Schannel clients (curl.exe, PowerShell) do not trust the MITM CA. **M** | Method/path-ruled hosts fail closed for Schannel tools; documented |
| R4 CA | **S** | **S** | **S** | Trust passed via `SSL_CERT_FILE`, `NODE_EXTRA_CA_CERTS`, `REQUESTS_CA_BUNDLE`, `GIT_SSL_CAINFO`; the file holds the public cert only |
| R5 sockets | `network-outbound (remote unix-socket)` denied; Apple Events off. **S** | Socket paths not mounted; seccomp denies `socket(AF_UNIX)` (socketpair allowed) and `io_uring`; netns isolates abstract sockets; Landlock Unix-socket scoping where the kernel has it. **S** | AppContainer SID needs a pipe DACL entry. **M (unverified)** | Probes connect to `docker_engine`, `openssh-ssh-agent` and an Everyone-ACL test pipe (all must fail) |
| R6 limits | Watchdog; `killpg` + descendant sweep; no memory cap. **W** | PID namespace; cgroup v2 via `systemd-run --user --scope` (`TasksMax`, `MemoryMax`) where delegated, else `RLIMIT_NPROC`. **S** kill / **M** limits | MXC kill-on-close job plus an outer job (`ActiveProcessLimit`, `JobMemoryLimit`). **S** (spike S-W3) | macOS: orphans stay sandboxed and `lyra-egress` revokes the session token at kill, so they lose network; hard limits only in T2; UI shows CPU and RSS |
| R7 env | `execve` with an allowlisted env. **S** | `bwrap --clearenv` + allowlist. **S** | Always an explicit, non-empty env block. **S** | — |
| R8 | Same launcher. **S** | **S** | **S** | `lyra-sandbox` file-tool worker mode |
| R9 | Probe suite. **S** | + userns / AppArmor / LSM probe. **S** | + MXC `probe()` must return exactly `base-container`; no AppContainer fallback. **S** | All: refuse T1 on any probe success |

Residual non-parity, stated plainly in the UI: process limits and tree kill on macOS (W); MXC alias and Schannel caveats
on Windows (M). T2 is the remedy for workloads that need hard limits.

**Linux packaging.** Lyra bundles bubblewrap 0.13.1 (≥ 0.12.0 for CVE-2026-87766), non-setuid, at
`/opt/Lyra/resources/bin/lyra-bwrap`, and never falls back to a system bwrap. The .deb/.rpm installs
`/etc/apparmor.d/lyra-bwrap`:

```
abi <abi/4.0>, profile lyra-bwrap /opt/Lyra/resources/bin/lyra-bwrap flags=(unconfined) { userns, }
```

Lyra never asks the user to set `kernel.apparmor_restrict_unprivileged_userns=0`.

**Windows floor.** MXC needs Windows 11 24H2 26100.9278+, 25H2 26200.9278+, 26H2 26300.9550+ or 26H1 28000.2804+;
.9550+ is recommended because earlier builds cannot host loopback servers inside the container. Older builds get
`sandbox_unsupported` and the coding agent is disabled, not weakened. `mxc-sdk` is pinned and built with
`MXC_TELEMETRY_COMPONENT_PATH=""`.

### 4.1 Unsigned dev builds

| OS | Behaviour |
|---|---|
| macOS | Seatbelt works unchanged with ad-hoc signing |
| Linux | Running from a build tree is outside `/opt/Lyra`, so the shipped AppArmor profile does not apply. On distros that restrict userns (Ubuntu 23.10+), `xtask dev-apparmor` installs a profile for the dev bwrap path (needs sudo once); otherwise T1 reports `sandbox_unsupported` with that instruction |
| Windows | MXC works unpackaged. If S-W2 shows the strict mode needs package identity, dev builds MAY use the identity-less mode, labelled "weaker network isolation (development build)" in the UI; release builds refuse it |

## 5. Egress proxy (`lyra-egress`)

One process per session, outside any sandbox, the same Rust binary on all three OSes (hyper, rustls, rcgen,
hickory-resolver). Its specification reuses two open-source designs: the resolved-address guard of
[`anthropic-experimental/sandbox-runtime`](https://github.com/anthropic-experimental/sandbox-runtime) (resolve once,
drop denied addresses, dial the survivor) and the domain-matcher rules of [`openai/codex`](https://github.com/openai/codex)
(no global `*`; `*.` vs `**.`). Neither of those projects issues a name-constrained, memory-only CA; Lyra does.

| Behaviour | Rule |
|---|---|
| Auth | Sandboxed clients present a per-session token; core revokes it when the session's processes are killed |
| CONNECT | Allowed only to listed hosts, with an SNI == Host check; opaque tunnels are refused |
| MITM | Only for hosts whose rules carry a method or path limit. A fresh CA per session, `nameConstraints.permittedSubtrees` = exactly those hosts, key in memory, destroyed at exit |
| DNS | Resolved by the proxy; private, loopback, link-local, CGNAT and metadata ranges (e.g. `169.254.169.254`) denied after resolution; the connected IP is pinned |
| Credentials | None injected in v1 |
| Audit | Every allow and deny decision is logged with session, host, method, path (when visible) and rule |
| Failure | Any parse or policy error fails closed |

Fallback if spike S-E1 fails: the sandbox-runtime proxy as a child process, patched for the CA.

## 6. Preflight probes

Run on first use per OS version, on every release in CI, and after any OS build change. T1 is refused if any probe succeeds.

| Probe | All OSes | Extra |
|---|---|---|
| Write outside roots | yes | Windows: write through a hard link to a protected file |
| Read `~/.ssh/id_*` and Lyra's data dir | yes | Windows: `%USERPROFILE%\.ssh`, `%APPDATA%\gh` |
| Connect to `1.1.1.1:443` directly | yes | |
| Connect to a non-proxy loopback port | yes | |
| Resolve a name without the proxy | yes | |
| Connect to docker and ssh-agent sockets/pipes | yes | Windows: an Everyone-ACL test pipe |
| Fetch `169.254.169.254` and `127.0.0.1:<port>` through the proxy | yes | |
| Spawn an escaping daemon (`setsid` + double fork), then check it is killed or loses network | yes | |
| macOS | `osascript` (Apple Events), `security` keychain reads | |
| Linux | userns available, AppArmor profile loaded, Landlock ABI and LSM list, cgroup delegation | |
| Windows | MXC `probe()` returns `base-container`; MXC telemetry not routed (ETW check) | |

## 7. Permission engine

**Modes** (per thread, recorded as events): `read-only` · `ask` (default) · `auto-edit` (workspace edits automatic,
commands ask) · `auto` (rules + sandbox + advisory judge) · `plan` (no side effects). There is no bypass mode outside T2.

**Rules:** `Tool(pattern)` with `allow` / `ask` / `deny`; deny wins; layered as in [architecture.md](architecture.md) §8.
Examples: `Shell(git status)`, `Shell(npm test:*)`, `Fs.write(src/**)`, `Mcp(github:create_pull_request)`, `Browser.transmit(*)`.

**Command analysis.** Shell commands are parsed with native `tree-sitter-bash` into simple commands, pipelines,
redirections and substitutions; a parse error fails closed. A rule matches only if **every** simple command matches.
Wrappers (`sh -c`, `bash -c`, `eval`, `xargs`, `env`, `nohup`, `ssh`, `docker run`, interpreters with `-c`/`-e`) are
opaque and require approval unless the tier is T2. The open-source Codex `execpolicy` is a design reference.

**Action classes.** Every tool declares one of `read`, `write-local`, `exec`, `transmit` (network side effects: push,
publish, send, post), `destructive` (delete, force, drop, reset), `financial`, `permission` (OS or account permission
changes). Floors:
- `transmit`, `destructive`, `financial` and `permission` are always `ask` in `ask` and `auto-edit`. In `auto` they need
  an explicit user `allow` rule; **a judge alone never approves them.**
- Unattended or background runs get `read`, `write-local` and `exec` in T2 only, unless the user pre-approved rules.

**Judge (advisory; after deterministic rules ship).** A provider-neutral pre-filter on the `judge` tier annotates each
pending action with risk, reversibility and injection signals. In `auto` mode it MAY approve `read`, `write-local` and
`exec` inside T1. "Unsure" or judge failure escalates to the human.

**Escalation ladder.** When a command fails because of the sandbox, the agent MAY request a strictly wider scope
(e.g. "allow network to `api.stripe.com` for this thread") with a justification. A native dialog shows the policy
diff; only the user approves. There is no `dangerouslyDisableSandbox`-style parameter.

**Checkpoints.** Before each write turn, touched files are snapshotted into `checkpoints/` (agent-unreadable,
write-denied); `/rewind` restores files and thread state.

## 8. Acceptance

| Test | Pass condition |
|---|---|
| SB-1 | All preflight probes are denied on every supported OS build (R9) |
| SB-2 | 30 real developer commands under the default policy need ≤ 6 non-narrow escalations (spike S2) |
| SB-3 | A corpus of 50 approval-bypass strings (chaining, wrappers, substitutions) all require approval (L-05) |
| SB-4 | `git push` to an arbitrary remote from the sandbox fails (L-37) |
| SB-5 | `docker run -v /:/host` from the sandbox fails (L-38) |
| SB-6 | An agent edit of `.lyra/policy.json` or `.git/hooks/pre-commit` fails (L-24) |
| SB-7 | A guest-written `.git/hooks/pre-commit` in T2 never reaches the host (L-39) |
| SB-8 | An off-list name issued under the session CA does not validate |
| SB-9 | Tool schemas expose no sandbox-disable parameter (L-20) |

Spikes: S2, S-M1, S-M2, S-L1, S-L2, S-W1, S-W2, S-W3, S-E1, S-VM ([spikes.md](spikes.md)).
