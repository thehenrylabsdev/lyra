# Contributing to Lyra

Thank you for helping. Lyra is pre-alpha, and the workflow below keeps it reviewable and safe.

## Before you start

1. Read [CLAUDE.md](CLAUDE.md). It applies to humans too: repo map, security invariants, workflow and quality gates.
2. Find or open an issue. Work is planned in [docs/plan/ROADMAP.md](docs/plan/ROADMAP.md) as epics → stories → tasks
   → subtasks. **One subtask is one pull request.**
3. For a design change, open an issue first. Decisions are recorded as ADRs in [docs/adr/](docs/adr/).

## Development setup

- Rust: install `rustup`. The toolchain is pinned in `rust-toolchain.toml` and installs automatically.
- Local tools: `cargo install --locked cargo-nextest cargo-llvm-cov cargo-deny cargo-audit cargo-mutants`, plus Docker
  for the Linux job. `cargo xtask doctor` checks them.
- Node.js and pnpm are needed for the desktop app once it exists. Versions are pinned in `package.json`.

## Making a change

```sh
git switch -c e1/1.2.1.a-short-slug
# ...edit, with tests...
cargo xtask ci                 # fmt, clippy, tests, coverage, deny, audit
cargo xtask mutants --diff     # every changed line of logic must be caught by a test
cargo xtask docker-ci          # the Linux job in Docker
git commit -s -m "feat(core): short summary"
```

- **Tests are required for every logic change.** Mutation testing on the diff must leave no surviving mutants.
- **Commits** use [Conventional Commits](https://www.conventionalcommits.org/) and the
  [Developer Certificate of Origin](https://developercertificate.org/) sign-off (`-s`). We do not use a CLA.
- **Dependencies:** crates.io only, at least 7 days old, and allowed by `deny.toml`. Say why in the PR.
- **Never** commit secrets, `.env` files, credentials or generated files, except the checked-in protocol types.

## Pull requests

- Title `[<subtask id>] Imperative summary`; fill in the template.
- CI must be green on Linux, macOS and Windows.
- PRs are squash-merged into `main`.

## Code of conduct

This project follows the [Contributor Covenant](CODE_OF_CONDUCT.md).
