# Implementation Plan: Read-Only Browsing of a Devtron Instance

**Branch**: `001-readonly-browsing` | **Date**: 2026-09-24 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-readonly-browsing/spec.md`

## Summary

A keyboard-driven terminal UI in the style of k9s, used to browse registered Devtron instances
without changing anything on them. It covers the application list, per-environment deployment
state, deployment history with pre/post stages, CI builds, and live logs for builds, stages and
pods, with switching between named instances.

How it is built:

- **Two crates in a Cargo workspace.**
  - `devtron-api` is the HTTP client. It has no terminal dependencies, uses hand-written tolerant
    `serde` models checked against live responses from the reference instance, and enforces a read-only
    **allowlist** of endpoints.
  - `devtron-tui` is the terminal app. Its message loop is in the style of the Elm architecture.
- **Instance switches** cancel all in-flight work, and messages carry an **epoch** so late results
  from the previous instance are dropped.
- **All logs** share one server-sent-events pipeline and one log viewer.

Endpoint choices and the facts behind them are in [research.md](research.md).

## Technical Context

**Language/Version**: Rust stable, MSRV 1.90 (local toolchain 1.90). Highest dependency MSRV is
ratatui's 1.88.

**Primary Dependencies**:

- In the constitution's list: `ratatui` 0.30, `crossterm` 0.29, `tokio` 1.53, `reqwest` 0.13
  (`rustls`), `serde`/`serde_json`, `clap` 4.6, `color-eyre` 0.6.
- Additional, justified in Complexity Tracking: `eventsource-stream`, `ansi-to-tui`, `tokio-util`,
  `futures`, `secrecy`, `base64`, `toml`, `directories`, `jiff`.

**Storage**:

- A local TOML config at `~/.config/devtron-tui/config.toml` holding the instance registry and
  `last_instance`.
- Tokens stay in their own files.
- No cache is kept on disk.

**Testing**: `cargo test`. Three kinds of test:

- Fixture deserialization tests: scrubbed recordings from the reference instance.
- `wiremock` HTTP tests: allowlist, errors, SSE, reconnects, switch races.
- `insta` snapshots of views rendered with `ratatui`'s `TestBackend`.

No test contacts a live instance.

**Target Platform**: Linux x86_64 terminals, including WSL2. Minimum 80×24, with colour.

**Project Type**: Terminal application (TUI) plus a reusable API client library.

**Performance Goals**:

- Filtered app list within 3 s of launch (SC-001).
- A filter keystroke re-renders in under 16 ms at 200 apps.
- Log lines appear within 2 s (SC-003).
- A switch completes within 5 s (SC-010).

**Constraints**:

- The render loop never waits on I/O (constitution III).
- Log buffers are capped at 50,000 lines per viewer.
- At most 4 concurrent resource-tree requests per app detail.
- Zero requests outside the allowlist (SC-005).

**Scale/Scope**:

- Instances of hundreds of apps. The reference instance today has 37 apps, 112 app-environment pairs
  and 6 projects.
- 9 views: instances, apps, app detail, history, deployment, builds, pods, log viewer, help.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

| Principle / rule | How this plan complies | Status |
|---|---|---|
| **I. Safe by default** | No mutating action exists in this feature. Reads are limited to an allowlist enforced before any request is sent (R3), with a test proving it. `--readonly` exists and is always on for now. The instance name and colour are always in the header. The resource tree is never requested when a deletion is pending (R3). | ✅ |
| **II. Credentials never leak** | Tokens are held as `secrecy::SecretString` (redacted `Debug`) and resolved per instance (env var → token file → default). There is no CLI flag for tokens. A permission warning fires on `mode & 0o077`. Each `Session` owns a `reqwest::Client` bound to its instance's token and base URL, so a token cannot reach another host. Fixtures are scrubbed, and a deny-list test enforces it (R12). The expiry is decoded from `exp` (R1). | ✅ |
| **III. Responsive UI** | Pure `update` plus async `Cmd`s over `mpsc` (R11). Every view has a `Loadable<T>` state (loading, loaded, empty, error). A panic hook restores the terminal. Log ring buffer capped at 50,000 lines. Switching cancels through `CancellationToken` and filters by epoch. | ✅ |
| **IV. Keyboard-first** | One key grammar across views (`contracts/keybindings.md`). A footer shows the keys of each view, and `?` opens help. Headless commands are out of scope for this feature (backlog), so there is no `-o json` yet. | ✅ |
| **V. Isolated, tolerant API layer** | `devtron-api` is its own crate with no `ratatui`/`crossterm` dependency, which Cargo enforces. Models are hand-written with `#[serde(default)]`, and unknown fields are ignored. Every endpoint has a fixture test. No live instance is needed in tests. | ✅ |
| Tech & constraints | The listed stack is used. Additional crates are justified below. No `unsafe` (`#![forbid(unsafe_code)]` in both crates). Linux target. Config follows the constitution's registry shape. HTTP API only, no kubectl. | ✅ with justified additions |
| Workflow gates | `fmt --check`, `clippy -D warnings` and `test` gate every commit. Fixtures and tests come with each endpoint. There is no mutating action, so the confirmation and read-only tests are not triggered yet. | ✅ |

**Post-design re-check (after Phase 1)**: still passing.

- The data model and contracts add no endpoint outside the allowlist.
- The only `POST` is `/app/list`, a pure query (R3).
- The resource-tree side effects are documented, and the deletion-pending case is avoided.

## Project Structure

### Documentation (this feature)

```text
specs/001-readonly-browsing/
├── spec.md
├── plan.md              # this file
├── research.md          # Phase 0: endpoint and design decisions (R1–R13)
├── data-model.md        # Phase 1: domain types, states, validation
├── quickstart.md        # Phase 1: how to validate the feature end to end
├── contracts/
│   ├── devtron-api.md   # consumed HTTP endpoints = the read-only allowlist
│   ├── keybindings.md   # the UI contract: views, keys, navigation
│   └── cli-config.md    # command-line flags, config file, token resolution
└── tasks.md             # Phase 2 (/speckit-tasks)
```

### Source Code (repository root)

```text
Cargo.toml                         # workspace: members, shared lints, MSRV 1.90
crates/
├── devtron-api/                   # no terminal dependencies (constitution V)
│   ├── src/
│   │   ├── lib.rs
│   │   ├── client.rs              # Client: base URL, token header, typed methods only
│   │   ├── allowlist.rs           # (method, path pattern) table + check before send
│   │   ├── auth.rs                # SecretToken, JWT exp/email decoding, perm check
│   │   ├── error.rs               # ApiError: Unreachable/Unauthorized/Expired/Forbidden/…
│   │   ├── envelope.rs            # {code,status,result,errors} parsing (R2)
│   │   ├── models/                # apps.rs, env.rs, history.rs, ci.rs, tree.rs, roles.rs
│   │   └── logs/                  # sse.rs (LogStream, LogEvent), stage_info.rs
│   ├── examples/record.rs         # dev-only recorder + scrubber → fixtures (R12)
│   └── tests/
│       ├── fixtures/              # scrubbed, committed
│       ├── models.rs              # one deserialization test per endpoint
│       ├── allowlist.rs           # every method hits only allowlisted routes
│       ├── errors.rs              # 401/403/expired/unreachable classification
│       ├── sse.rs                 # framing, END, reconnect with Last-Event-ID, pod close
│       └── fixtures_are_scrubbed.rs
└── devtron-tui/
    ├── src/
    │   ├── main.rs                # clap, color-eyre, terminal guard, run loop
    │   ├── config.rs              # registry load/save (atomic), token resolution
    │   ├── session.rs             # Session {client, cancel, epoch}; switch()
    │   ├── app/                   # state.rs, msg.rs, update.rs, cmd.rs (runner)
    │   ├── views/                 # instances, apps, app_detail, history, deployment,
    │   │                          # builds, pods, logs, help, register
    │   ├── widgets/               # header, footer, table, filter_bar, confirm, loadable
    │   ├── logbuf.rs              # ring buffer, search index, follow mode
    │   └── theme.rs               # colours, instance colour
    └── tests/
        ├── update.rs              # pure state-transition tests (navigation, filter, epoch)
        ├── switch_race.rs         # SC-011: switch mid-stream / mid-request (wiremock)
        ├── snapshots.rs           # insta snapshots of each view × loading/empty/error
        └── token_never_rendered.rs# SC-006: search all rendered buffers for the token
```

**Structure Decision**:

- Two crates in one Cargo workspace. The split makes Cargo enforce constitution V: `devtron-api`
  cannot import `ratatui` because it isn't a dependency.
- `CLAUDE.md`'s layout section (currently `Cargo.toml, src/, tests/`) is updated to this layout by
  the first implementation task.

## Complexity Tracking

> Dependencies outside the constitution's stack list, as the constitution requires.

| Addition | Why needed | Simpler alternative rejected because |
|---|---|---|
| `eventsource-stream` | All log views are SSE (R8). | Parsing SSE by hand repeats a spec with edge cases (multi-line `data`, `id:` with or without a space, comments). `reqwest-eventsource` is stale and pinned to an old reqwest. |
| `ansi-to-tui` | Build logs contain ANSI colour codes (R8). | Stripping codes loses the red/green that makes failures readable. Converting them by hand is a small parser we would have to maintain. |
| `tokio-util` | `CancellationToken` for switching (FR-033, R11). | A hand-rolled `watch` channel plus `select!` in every task is the same thing, less tested. |
| `futures` | Stream combinators for the SSE and paging streams. | `tokio-stream` covers less. Hand-written `poll_next` is error-prone. |
| `secrecy` | Tokens that redact themselves (constitution II). | A custom newtype with a manual `Debug` works but doesn't zero memory on drop, and is easy to leak through `Display`. |
| `base64` | Decoding the JWT payload for `exp` (R1). | A JWT crate brings signature code we can't use (no secret). |
| `toml` | Config file (R13). | serde has no TOML support built in. |
| `directories` | XDG config path (R13). | Hard-coding `~/.config` ignores `XDG_CONFIG_HOME`. |
| `jiff` | Parsing Devtron timestamps (RFC 3339 with zone) and formatting ages. | `std` has no calendar or time-zone support. `chrono` is the older alternative with more pitfalls around time zones. |
