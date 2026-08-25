# managed-codex-agent

A harness that turns a Codex CLI profile into a content-addressed bundle,
installs it into an isolated managed home, verifies the result before spending a
turn, and then drives `codex app-server` over JSON-RPC with every byte it prints
bounded and escaped.

**Experimental. No support guarantee.** This is a reference implementation
published for reading, not a product. See [SECURITY.md](SECURITY.md).

## Why it exists

Running a coding agent against a real repository is easy to do once and hard to
do repeatedly. The interesting failures are not in the model: they are in the
state the previous run left behind, in output that a terminal will happily
interpret as control sequences, and in diagnostics that say something went
wrong without saying what.

This project is an answer to that shape of problem.
[docs/dogfooding-case-study.md](docs/dogfooding-case-study.md) records what two
real runs found and what changed between them.

## Support boundaries

Read these before installing. They are deliberate, and each one is enforced
rather than documented:

- **macOS only.** `resolveRuntimePaths` refuses any other platform.
- **One exact Codex version**, pinned in `src/constants.ts` as
  `REQUIRED_CODEX_VERSION`. The generated protocol contract is only known to
  match that version, so a mismatch is refused rather than attempted.
- **The target repository must be a clean worktree**, and so must the bundle
  source. Submodules and gitlinks are rejected.
- **One run at a time**, guarded by a single process lock.

The npm package stays `"private": true` so it is never published to a registry
by accident. The GitHub repository being public and the package being private
are unrelated settings.

The package name is `andrew-code-agent` and the repository is
`managed-codex-agent`. The package name feeds the managed state path, the CLI
usage strings, and the bundle digest frame, so renaming it would invalidate
every recorded digest; it is left alone deliberately.

## Requirements

- macOS
- Node.js at the version in `.nvmrc`
- pnpm at the version in `package.json`'s `packageManager`
- A Codex CLI install at the pinned version
- A Codex profile directory that is a clean Git worktree with every bundled
  file tracked

## Install

```bash
pnpm install --frozen-lockfile
pnpm build
```

All test files import the compiled output, so build before testing.

## Configure

Four environment variables define the whole runtime surface:

- `ANDREW_AGENT_CODEX_SOURCE` — bundle source root, which must be a clean Git
  worktree root. Defaults to `$HOME/.codex`.
- `ANDREW_AGENT_STATE_ROOT` — managed runtime state root. Defaults to
  `$HOME/Library/Application Support/andrew-code-agent`.
- `ANDREW_AGENT_CODEX_BIN` — absolute path to the `codex` executable.
  Otherwise resolved from `PATH`.
- `ANDREW_AGENT_ORACLE_ROOT` — knowledge base root for the `oracle`
  capability. Unset leaves the capability off.

The source root needs an `agent-bundle.toml` describing what to bundle. See
[`test/fixtures/manifests/valid.toml`](test/fixtures/manifests/valid.toml) for a
complete example with every section populated.

The two roots must not overlap, and neither may overlap the Oracle root. State
directories must be owner-only `0700` ordinary directories.

## Use

```bash
node dist/cli.js doctor
node dist/cli.js run /path/to/repository "the task"
node dist/cli.js resume <thread-id> "the next task"
node dist/cli.js status
```

`doctor` reports readiness without spending a turn, including which capability
the current environment would enable. `status` with no argument resolves the
latest thread for the repository you are standing in. `resume` with no prompt
prints the stored record and starts no turn.

Exit codes:

- `0` — succeeded, or the turn completed.
- `1` — the turn failed, the thread was not found, or output could not be
  written.
- `2` — invalid command usage.
- `3` — preflight or runtime preparation failed, including a Doctor blocker.
- `4` — the App Server failed.
- `130` — the turn was interrupted.

## Capabilities

A capability is an optional slice of the bundle — files, hooks, and instruction
sections rendered only when it is enabled, plus external executables asserted
present before it is.

`oracle` is the only one here. It is **off by default** and needs a knowledge
base you supply:

```bash
ANDREW_AGENT_ORACLE_ROOT=/path/to/your/knowledge-base \
  node dist/cli.js run --capability oracle /path/to/repository "the task"
```

Requesting it without that variable fails by name rather than quietly running
without it. The root's location is read from the environment rather than a flag
so it stays out of shell history, and only a digest of it is recorded, never the
path.

There is no bundled knowledge base. What `oracle` is worth depends entirely on
what you point it at, which is why it is a capability rather than a feature.

## Verify

```bash
pnpm check        # typecheck plus all four test layers
```

Three gates are opt-in and skip with a printed notice when their environment is
absent, so a default green `pnpm check` does not cover them:

- `test/contract/client.test.mjs` needs `ANDREW_AGENT_PINNED_CODEX_BIN`, and
  proves the committed contract matches what the pinned Codex generates, byte
  for byte. A mismatched version fails rather than skips.
- `test/integration/live-manifest.test.mjs` needs `ANDREW_AGENT_CODEX_SOURCE`,
  and proves a real bundle source produces one deterministic digest.
- The live smokes in `test/e2e/acceptance.test.mjs` need
  `ANDREW_AGENT_REAL_SMOKE` and `ANDREW_AGENT_SMOKE_CODEX_BIN`, plus
  `ANDREW_AGENT_SMOKE_AUTH` for the two-run gate. They prove a real Codex
  accepts the installed configuration, and that a second run survives the
  configuration rewrite the first one caused.

Each layer script is wrapped so a missing directory exits 0, and a skipped test
counts as a pass. **Read a green aggregate as "nothing failed", not as
"everything ran."**

## Documentation

- [docs/architecture.md](docs/architecture.md) — what a run crosses, layer by
  layer
- [docs/threat-model.md](docs/threat-model.md) — what is trusted, what is not,
  and where each boundary sits
- [docs/dogfooding-case-study.md](docs/dogfooding-case-study.md) — two real runs
  and what they found

## Generated contract

`src/generated/codex-app-server/**` and `schemas/codex-app-server/**` are
produced by Codex's own generators at the pinned version. Never edit or format
them by hand:

```bash
codex app-server generate-ts --out src/generated/codex-app-server
codex app-server generate-json-schema --out schemas/codex-app-server
```

The generated TypeScript package uses the CommonJS boundary in
`src/generated/package.json`, which the build copies to
`dist/generated/package.json`.

See [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) for upstream attribution.

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
