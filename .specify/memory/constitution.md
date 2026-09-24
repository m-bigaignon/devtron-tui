
# Devtron TUI Constitution

## Core Principles

### I. Safe by Default (NON-NEGOTIABLE)

The tool operates on production deployments, so no action may change Devtron state by accident.

- Every view MUST be read-only unless the user deliberately invokes a mutating action.
- Every mutating action (build trigger, deploy, rollback, pod rotation, config change, delete)
  MUST go through a confirmation dialog that names the exact instance, app, environment and action.
  The default choice in that dialog MUST be "cancel".
- A single keystroke MUST NOT perform a mutation on its own.
- A `--readonly` mode MUST exist and MUST hide or disable every mutating action, both in the
  TUI and in headless subcommands.
- A bulk mutation MUST list every affected target before it is confirmed.
- The active instance's name MUST be visible at all times. Users MUST be able to give each instance
  a colour (for example red for production), shown in the header and in every confirmation dialog.

Rationale: k9s-style speed is only acceptable if a mistaken keypress cannot redeploy production.

### II. Credentials Never Leak (NON-NEGOTIABLE)

- Each registered instance MUST have its own token file (default `~/.devtron-token`).
  `DEVTRON_TOKEN` MAY override the token of the active instance only, and MUST NOT be sent to any
  other instance. A token MUST NOT be accepted as a command-line argument, which would leak
  through shell history and process listings, and MUST NOT be stored in the configuration file.
- A token MUST only ever be sent to the instance it belongs to.
- The tool MUST warn when the token file is readable by group or others.
- The token MUST NOT appear in logs, error messages, panic output, `Debug` output, the screen
  or test fixtures. Types holding it MUST redact it in their `Debug` implementation.
- Recorded API fixtures MUST be scrubbed of tokens, emails and internal hostnames before
  they are committed.
- The tool MUST decode the token's `exp` claim, show the expiry, and report an expired token
  explicitly rather than as a generic 401.

Rationale: an API token for a CI/CD platform effectively has deploy rights over every
connected cluster.

### III. Responsive, Never-Blocking UI

- The render loop MUST NOT wait on network or disk I/O. All Devtron calls run in tokio tasks
  and send their results back to the UI thread over channels.
- Every data view MUST have explicit loading, empty and error states. A failed request MUST
  NOT crash the TUI or leave the screen blank.
- The terminal MUST be restored (raw mode off, alternate screen left, cursor shown) on normal
  exit, on error and on panic.
- Log panels MUST stream incrementally and MUST limit how much output they keep in memory.
- Only one instance is active at a time. Switching instance MUST cancel every request and stream
  of the previous instance and discard its data, so no view shows a mix of instances.

Rationale: a slow or unreachable Devtron instance must degrade the tool, not freeze it.

### IV. Keyboard-First, Consistent Navigation

- Navigation MUST follow one grammar across all views: `:` switches views, `/` filters,
  `Enter` opens the selected item, `Esc` goes back, `?` shows help, `q` quits.
- Each view MUST show the key bindings available in it (footer hints) and MUST NOT rebind
  the global keys above.
- Every feature MUST be reachable from the keyboard. Mouse support is optional.
- Where a headless subcommand exists for a view, it MUST support `-o json`, write data to
  stdout and diagnostics to stderr, and exit non-zero on failure.

Rationale: the tool is only faster than the web UI if its controls become muscle memory.

### V. Isolated, Tolerant API Layer

- The Devtron client MUST be a separate module (or crate) with no dependency on
  `ratatui` or `crossterm`, so it can be tested and reused without a terminal.
- API models MUST be hand-written `serde` types, limited to the endpoints the tool uses,
  with `specs/` from the Devtron repository as the reference. Generating a client from the
  full spec set is out of scope unless an amendment says otherwise.
- Deserialization MUST ignore unknown fields and use defaults for missing optional ones.
  A missing field MUST NOT abort parsing of a whole list.
- Every endpoint the tool uses MUST have a deserialization test against a scrubbed response
  recorded from a real instance. Tests MUST NOT require a live Devtron instance.

Rationale: Devtron's specs are hand-maintained and drift from what the server actually
returns. Real fixtures catch that drift. Strict parsing would turn it into outages.

## Technology & Constraints

- Language: Rust, stable toolchain, minimum supported version 1.90. `unsafe` code is
  forbidden unless an amendment justifies it.
- TUI: `ratatui` with the `crossterm` backend. Async: `tokio`. HTTP: `reqwest` with `rustls`
  (no OpenSSL dependency). Serialization: `serde`. CLI: `clap`. Errors: `color-eyre`.
- Adding a runtime dependency outside this list MUST be justified in the feature's plan.
- Target platform: Linux x86_64, including WSL2. Other platforms are welcome but not required.
- Delivery: a single binary. Configuration lives at `~/.config/devtron-tui/config.toml`: a registry
  of named instances (address, token file path, colour) and preferences. Secrets never go in
  that file (Principle II). Views combining several instances at once are out of scope.
- Devtron is reached only through its HTTP API (`/orchestrator/...`). The tool MUST NOT
  require kubectl access or direct cluster credentials.

## Development Workflow

- One repository holds the governance (`.specify/`), the specs (`specs/`), the backlog
  (`backlog.md`) and the Rust crate at its root. Spec changes and the code that implements
  them MAY land in the same commit.
- Features follow the spec-kit cycle: specify, (clarify), plan, tasks, implement. The plan's
  Constitution Check MUST pass, or its violations MUST be listed in Complexity Tracking.
- Quality gates before merge: `cargo fmt --check`, `cargo clippy -- -D warnings` and
  `cargo test` MUST all pass.
- A change adding or changing an endpoint MUST come with its scrubbed fixture and test
  (Principle V). A change adding a mutating action MUST come with a test showing that it
  requires confirmation and is disabled in `--readonly` mode (Principle I).
- Code, commit and documentation conventions are defined in `CLAUDE.md` at the repository root.

## Governance

This constitution overrides any other practice in this repository. Where
the two conflict, the constitution wins until it is amended.

- Amendments are made by a pull request to this repository. The request states what
  changes, why, and which existing specs or code are affected.
- Versioning follows semantic versioning. MAJOR: a principle is removed or redefined in an
  incompatible way. MINOR: a principle or section is added or materially expanded. PATCH:
  wording and clarifications.
- Every plan and every code review MUST check compliance with the principles above. Any
  exception MUST be written into the plan's Complexity Tracking with its justification.
  An undocumented exception is a defect.

**Version**: 1.1.0 | **Ratified**: 2026-09-24 | **Last Amended**: 2026-09-24
