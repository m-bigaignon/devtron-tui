---

description: "Task list for feature 001: read-only browsing of a Devtron instance"
---

# Tasks: Read-Only Browsing of a Devtron Instance

**Input**: Design documents from `specs/001-readonly-browsing/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: Required. The constitution asks for three kinds:

- Principle V: a fixture deserialization test for every endpoint.
- Principle II: a scrub check on committed fixtures.
- Development Workflow: `fmt`, `clippy -D warnings` and `test` gates.

The plan and quickstart name the tests that carry the spec's guarantees: allowlist (SC-005),
switch race (SC-011) and token never rendered (SC-006).

**Organization**: Tasks are grouped by user story so that each story can be implemented and
tested on its own.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on incomplete tasks)
- **[Story]**: User story from spec.md (US1–US6)
- Paths are relative to the repository root. The workspace has two crates: `crates/devtron-api`
  (no terminal dependencies) and `crates/devtron-tui`.
- References in parentheses: `R#` = research.md, `A#` = contracts/devtron-api.md,
  `FR`/`SC` = spec.md.

## Standing rules for every task

- Wire models use `#[serde(default)]` and ignore unknown fields. Empty strings and `0` ids become
  `None` in the domain conversion (data-model.md, preamble).
- Every new endpoint method gets its row in `crates/devtron-api/src/allowlist.rs`, a scrubbed
  fixture, and a test in `crates/devtron-api/tests/models.rs` (constitution V).
- No committed file may contain the reference instance's address, organisation name, app names or
  user emails (CLAUDE.md).
- `cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test` passes at the end
  of every task.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Workspace, crates, toolchain and CI.

- [ ] T001 Create the workspace manifest `Cargo.toml`:
  - `[workspace] members = ["crates/devtron-api", "crates/devtron-tui"]`, `resolver = "3"`
  - `[workspace.package]`: `edition = "2024"`, `rust-version = "1.90"`, `license = "MIT"`,
    `repository = "https://github.com/m-bigaignon/devtron-tui"`
  - `[workspace.lints.rust] unsafe_code = "forbid"`
  - `[workspace.lints.clippy] all = "warn"`
- [ ] T002 [P] Create `crates/devtron-api/Cargo.toml` and `crates/devtron-api/src/lib.rs`
  (`#![forbid(unsafe_code)]`, `lints.workspace = true`).
  - Dependencies: `reqwest` 0.13 with rustls TLS, `json` and `stream` (no native-tls/OpenSSL),
    `serde` (derive), `serde_json`, `tokio` (rt, macros, time), `tokio-util`, `futures`,
    `eventsource-stream` 0.2, `secrecy` 0.10, `base64` 0.23, `jiff` 0.2.
  - Dev-dependencies: `wiremock` 0.6, `tokio` (full).
  - There MUST be no `ratatui` or `crossterm` dependency (constitution V).
- [ ] T003 [P] Create `crates/devtron-tui/Cargo.toml` and `crates/devtron-tui/src/main.rs` (hello-world
  placeholder, binary name `devtron-tui`, `#![forbid(unsafe_code)]`, `lints.workspace = true`).
  - Dependencies: `devtron-api` (path), `ratatui` 0.30, `crossterm` 0.29 (`event-stream`), `tokio`
    (full), `tokio-util`, `futures`, `clap` 4.6 (derive), `color-eyre` 0.6, `toml` 1.1,
    `directories` 6, `ansi-to-tui` 8, `secrecy` 0.10, `jiff` 0.2.
  - Dev-dependencies: `insta` 1.48, `wiremock` 0.6.
- [ ] T004 [P] Add `rustfmt.toml` (`max_width = 100`) and `.github/workflows/ci.yml`. The CI job runs
  on ubuntu-latest with Rust 1.90:
  - `cargo fmt --check`
  - `cargo clippy --all-targets -- -D warnings`
  - `cargo test`
  - `cargo tree -i openssl` must print nothing
- [ ] T005 [P] Update the Layout section of `CLAUDE.md` to list `crates/devtron-api/` and
  `crates/devtron-tui/` instead of `Cargo.toml, src/, tests/` (plan.md, Structure Decision).

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The API core (auth, errors, envelope, allowlist, client, SSE), the fixture pipeline,
the TUI message loop, the shared widgets and the shared log viewer. The log viewer is used by US3,
US4 and US5, so it is built here to keep those stories independent of each other.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

### API core (`crates/devtron-api`)

- [ ] T006 [P] Implement `crates/devtron-api/src/auth.rs` (data-model "Credentials"):
  - `Credentials` enum with one variant in this feature:
    `ApiToken { secret: secrecy::SecretString, claims: TokenClaims }`. `Debug` prints
    `Credentials(***)`, and there is no `Display`.
  - `fn apply(&self, headers: &mut HeaderMap)` sets the header for the variant (`token`, marked
    `set_sensitive(true)`). Future variants such as `SessionCookie` add a match arm here and nothing
    else.
  - `TokenClaims { expires_at: Option<jiff::Timestamp>, email: Option<String> }`, decoded from the
    JWT payload (second segment, base64 URL-safe without padding) with no signature check. A
    non-JWT token gives `None` fields (R1).
  - `fn is_readable_by_others(mode: u32) -> bool` returns `mode & 0o077 != 0` (FR-005).
  - Unit tests in the same file: `Debug` output, a fixed JWT's `exp` and `email`, a non-JWT token,
    mode `0o600` vs `0o644`, and that `apply` marks the header sensitive.
- [ ] T007 [P] Implement `crates/devtron-api/src/error.rs`, the `ApiError` enum from the
  classification table in `contracts/devtron-api.md`:
  - variants: `Unreachable { reason }`, `Expired { at }`, `Unauthorized`, `Forbidden { message }`,
    `NotFound`, `Server { status, message }`, `Decode { endpoint }`, `NotAllowed { method, path }`,
    `DeletionPending`
  - `fn retryable(&self) -> bool`
  - `Display` messages MUST NOT contain credentials, headers, or the query string. They say
    "credentials", not "token". A fix hint that depends on the auth kind (for example "replace
    {path}") is added by the caller from `AuthConfig::describe()`.
- [ ] T008 [P] Implement `crates/devtron-api/src/envelope.rs` (R2):
  - `Envelope<T> { code: Option<u16>, status: Option<String>, result: Option<T>, errors: Vec<WireError> }`
  - `WireError.userMessage` is a `serde_json::Value` (string or object).
  - `fn error_message(body: &[u8]) -> Option<String>` takes `errors[0].userMessage`, else `result`
    if it is a string, else `status`.
  - Unit tests on the three observed error bodies:
    - `{"code":401,"result":"Unauthorized"}`
    - `{"code":403,"status":"Forbidden","result":"unauthorized user"}`
    - `{"code":400,"errors":[{"code":"200","userMessage":"logs-not-stored-in-repository"}]}`
- [ ] T009 Implement `crates/devtron-api/src/allowlist.rs` (R3):
  - `Route` enum with one variant per row A1–A12 of `contracts/devtron-api.md`. Each variant has a
    method and a path pattern with `{}` segments.
  - `fn check(method: &Method, path: &str) -> Result<Route, ApiError::NotAllowed>`
  - Unit tests: every row matches its sample path; `POST` is allowed only for `/app/list`;
    `DELETE /app/list`, `PUT /team` and `/app/unknown` are refused.
- [ ] T010 Implement `crates/devtron-api/src/client.rs` (R1–R3). Depends on T006–T009.
  - `Client::new(base_url, Credentials) -> Result<Client>`:
    - strips a trailing `/` and a trailing `/orchestrator`
    - requires `https`, except for `localhost` and `127.0.0.1`
    - builds its own `reqwest::Client` with 20 s timeout, and default headers from
      `Credentials::apply`
    - one `Client` per instance, never shared (constitution II)
  - A private `async fn send<T>(&self, method, path, query, body) -> Result<T, ApiError>`:
    - checks the allowlist before sending
    - maps connection errors to `Unreachable`, and 401 to `Expired` when the claims' `expires_at`
      is in the past, else to `Unauthorized`
    - maps 403 to `Forbidden`, 404 to `NotFound`, and other ≥ 400 to `Server`, using
      `envelope::error_message`
  - The public `async fn check_roles() -> Result<Roles>` (A1) returns
    `Roles { roles: Vec<String>, super_admin: bool }`.
  - Re-export `Client`, `ApiError`, `Credentials` and `TokenClaims` from `lib.rs`.
- [ ] T011 [P] Implement `crates/devtron-api/src/logs/sse.rs` and
  `crates/devtron-api/src/logs/stage_info.rs` (R8):
  - `LogEvent = Line { id, text } | Stage(StageInfo) | End | Reconnecting { attempt } | Error(String) | Closed`
  - `LogStream` turns a `reqwest::Response` into `impl Stream<Item = LogEvent>` using
    `eventsource-stream`.
  - Both framings are handled:
    - CI/CD: `START_OF_STREAM`, `END_OF_STREAM` → `End`; `UNEXPECTED_END_OF_STREAM` →
      reconnect; `RECONNECT_STREAM` → ignored.
    - Pod: `id: <nanos>` with a space; `PING` → ignored; `CUSTOM_ERR_STREAM` → `Error`; a closed
      connection → `Closed`.
  - Lines starting `STAGE_INFO|` are parsed as `{stage, startTime, endTime, status}` → `Stage`.
  - A non-`text/event-stream` or non-200 response → `Error(message)` via `envelope::error_message`.
  - Reconnect: up to 3 attempts with backoff (1 s, 2 s, 4 s), sending `Last-Event-ID` with the last
    `id`. Every attempt is cancellable through a `CancellationToken` passed in.
- [ ] T012 [P] Create the fixture pipeline in `crates/devtron-api/examples/record.rs` (R12), with two
  subcommands:
  - `record --url <url> --token-file <path> --app <id> --env <id>` calls A1–A12 read-only and
    writes raw responses to `fixtures-raw/`.
  - `scrub` rewrites `fixtures-raw/*` into `crates/devtron-api/tests/fixtures/`:
    - emails → `user<n>@example.com`
    - the instance host and every pattern in `fixtures-raw/denylist.txt` →
      `devtron.example.com` / `example`
    - image registries → `registry.example.com`
    - git URLs → `https://git.example.com/repo<n>.git`
    - commit messages → `commit message <n>`
    - app, env and cluster names → `app<n>`/`env<n>`/`cluster<n>`
    - JWT-looking strings → dropped
    - SSE `data:` log text → `log line <n>`, keeping the framing, `id:` values and `STAGE_INFO`
      JSON
  - `scrub` also writes a synthesized `cd-stage-logs.sse` from `ci-logs.sse` and marks it with a
    first-line comment `: synthesized from ci-logs.sse (R6)`.
- [ ] T013 Run `cargo run -p devtron-api --example record -- scrub` over the existing `fixtures-raw/`.
  Rename `history-new.json` to `history.json` and drop `history-legacy.json`. Commit the scrubbed
  files to `crates/devtron-api/tests/fixtures/`:
  - `check-roles.json`, `app-list.json`, `team.json`, `other-env.json`, `detail-v2.json`,
    `resource-tree.json`, `history.json`, `ci-min.json`, `ci-workflows.json`
  - `ci-logs.sse`, `cd-stage-logs.sse`, `pod-logs.sse`

  Depends on T012.
- [ ] T014 [P] Write `crates/devtron-api/tests/fixtures_are_scrubbed.rs`. It fails if any file under
  `tests/fixtures/` contains:
  - an email outside `example.com`
  - a JWT-like string (`eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.`)
  - a hostname outside `*.example.com`
  - any line of `fixtures-raw/denylist.txt`, matched case-insensitively and only when that file
    exists
- [ ] T015 [P] Write `crates/devtron-api/tests/errors.rs` with wiremock. Each case yields the right
  `ApiError`, and its `Display` never contains the credentials (FR-003, SC-008):
  - unreachable (a closed local port)
  - 401 with a non-expired token → `Unauthorized`
  - 401 with an expired `exp` → `Expired`
  - 403 with a `result` string → `Forbidden`
  - 404 → `NotFound`
  - 500 with `errors[]` → `Server`
- [ ] T016 [P] Write `crates/devtron-api/tests/sse.rs` using `ci-logs.sse`, `pod-logs.sse` and
  `cd-stage-logs.sse` served by wiremock:
  - line count matches the fixture's `id:` frames
  - `STAGE_INFO` becomes `Stage` events
  - `End` arrives on `END_OF_STREAM`
  - the pod stream yields `Closed` on EOF
  - a mid-stream disconnect reconnects with the correct `Last-Event-ID` header, with no duplicate
    and no missing line
  - after 3 failed reconnects → `Error`
  - a 403 JSON response → `Error` with its message
  - latency (SC-003): a small `tokio::net::TcpListener` server (wiremock can't send a body
    incrementally) writes one SSE frame, waits 500 ms, then writes a second. The second
    `LogEvent::Line` must be received within 100 ms of being written.
- [ ] T017 Create the allowlist harness `crates/devtron-api/tests/allowlist.rs`:
  - a wiremock server that records every request, plus a helper `assert_all_allowlisted(&server)`
    that checks each received (method, path) against `allowlist::check`
  - a test that `Route`'s allowlist contains only `GET`, plus `POST` for `/app/list`
  - every story adds its methods to the test `every_public_method_stays_in_allowlist` in this file

  Depends on T009 and T010.
- [ ] T018 Create `crates/devtron-api/tests/models.rs` with a helper `load_fixture<T>(name)` and a
  test for A1 (`check-roles.json` → `Roles`). Each story adds its endpoints here. Depends on T013.

### TUI core (`crates/devtron-tui`)

- [ ] T019 [P] Implement `crates/devtron-tui/src/config.rs` (data-model InstanceConfig/Registry,
  contracts/cli-config.md):
  - The default path comes from `directories` (`$XDG_CONFIG_HOME/devtron-tui/config.toml`), and
    `--config` overrides it.
  - `name` must match `^[a-z0-9][a-z0-9-]{0,31}$`.
  - `url`: "`https` required (`http` only for `localhost`)"; a trailing `/` and `/orchestrator`
    are stripped.
  - `auth`: `AuthConfig::TokenFile { path }` from `auth = { kind = "token_file", path = "…" }`.
    `path` defaults to `~/.devtron-token` when `auth` is absent, and a leading `~` is expanded.
    Any other `kind` is refused with "auth kind '{kind}' is not supported by this version".
    `AuthConfig::describe()` returns for example "token file ~/.devtron-token" for messages.
  - `color`: a named colour or `#rrggbb`.
  - A key named `token`, `password` or `secret`, at any depth, is refused with a message saying
    why (the file never holds credentials).
  - Unknown keys are preserved on save (keep a `toml::Table` alongside the typed view).
  - Saving is atomic (write `config.toml.tmp`, then rename).
  - Unit tests for each rule.
- [ ] T020 [P] Implement `crates/devtron-tui/src/credentials.rs` (cli-config.md, "Credentials for an
  instance"):
  - `fn load(auth: &AuthConfig) -> Result<Loaded>`, where
    `Loaded { credentials: Credentials, perm_warning: Option<PathBuf> }`. For `TokenFile`, read the
    file, trim it, and decode the claims.
  - **No environment variable is read** (FR-002, constitution v1.2.0 II).
  - Errors: "cannot read token file {path}".
  - Unit tests: the permission warning fires on `0o644` and not on `0o600`; setting
    `DEVTRON_TOKEN` in the test environment changes nothing.
- [ ] T021 Implement the message loop in `crates/devtron-tui/src/app/` (R11):
  - `state.rs`: `State` with a view stack, `Session` summary and `Loadable<T>` =
    `Loading | Loaded(T) | Empty | Failed { cause, retryable }`, plus a `stale: bool` for refreshes
    (FR-022).
  - `msg.rs`: `Msg` with `epoch: u64` on every task result.
  - `update.rs`: `fn update(&mut State, Msg) -> Vec<Cmd>`, pure. It drops task messages whose
    epoch is not the current session epoch.
  - `cmd.rs`: a runner that spawns each `Cmd` on tokio with the session's `CancellationToken` and
    sends `Msg`s back over `mpsc`.
- [ ] T022 Implement `crates/devtron-tui/src/session.rs`:
  - `Session { instance: InstanceConfig, client: devtron_api::Client, cancel: CancellationToken, epoch: u64 }`
  - `fn switch(&mut self, new) -> Session` cancels the old token and sets `epoch += 1`.

  Depends on T019–T021.
- [ ] T023 Implement the terminal lifecycle in `crates/devtron-tui/src/main.rs` (FR-023, SC-007):
  - `clap` flags `--instance <name>`, `--config <path>`, `--readonly` (accepted, always on in this
    feature).
  - Install `color-eyre` and a panic hook that restores the terminal (raw mode off, alternate
    screen left, cursor shown) before the report prints. A `Drop` guard does the same on every
    return path.
  - Main loop: `tokio::select!` over crossterm `EventStream`, the `mpsc` receiver, a 250 ms tick,
    and `tokio::signal::unix` streams for SIGTERM and SIGHUP. Each signal leaves the loop the same
    way as a normal quit, so the terminal is restored (SC-007). Ctrl-C arrives as a key event in
    raw mode. SIGKILL cannot be caught and is out of scope.
  - The loop is a function
    `run(events: impl Stream<Item = Event>, backend: impl Backend, …) -> Result<()>` that `main`
    calls with the real crossterm stream and backend, so tests can drive it (T028).
  - Exit codes: 0 quit, 1 fatal, 2 usage.

  Depends on T021.
- [ ] T024 [P] Implement the shared widgets in `crates/devtron-tui/src/widgets/` and
  `crates/devtron-tui/src/theme.rs`, following contracts/keybindings.md, "Always-visible chrome":
  - `header.rs`: `⎈ {instance}` in the instance colour · email · `token until {date}` (red when
    expired or under 7 days) · breadcrumb · `READ-ONLY` badge
  - `footer.rs`: the view's keys, then `? help`
  - `loadable.rs`: a spinner with what is loading, an explicit empty sentence, an error with the
    cause and `r to retry`. `Failed { cause: Forbidden }` renders as "not visible with your
    access", with no retry hint. The same renderer works for one table cell, so a single row can be
    forbidden while the rest of the view loads (FR-035).
  - `too_small.rs`: below 80×24, show "terminal too small, 80×24 needed"
  - `table.rs`: a selectable table with a `/` filter bar, case-insensitive, updated on each
    keystroke. A `None` cell renders as `—` (spec edge case on missing fields).
  - `crates/devtron-tui/src/dialogs/confirm.rs`: a yes/no dialog whose default is **No**
- [ ] T025 Implement the global keys and command bar in `crates/devtron-tui/src/app/update.rs` and
  `crates/devtron-tui/src/views/help.rs`, following contracts/keybindings.md, "Global keys":
  - `:` with `apps`, `instances`/`ctx`, `help` and `q`, with Tab completion
  - `/`, `Enter`, `Esc` (back, does nothing on the root view, cancels a bar or dialog), `?`, `r`,
    `q` (**quits from any view**, constitution IV; it is plain text while typing in a bar or
    dialog field), `Ctrl-C` (quit)
  - `j`/`k`/arrows/`PgDn`/`PgUp`/`g`/`G`
  - The help view lists the global keys plus the key table of the current view (each view exposes
    `fn keys() -> &'static [(&str, &str)]`).

  Depends on T021 and T024.
- [ ] T026 [P] Implement `crates/devtron-tui/src/logbuf.rs` (data-model LogBuffer):
  - "Ring of at most 50,000 styled lines. On overflow the oldest is dropped and `dropped += 1`."
  - `follow` starts `true`, becomes `false` when the user scrolls up, and becomes `true` again at
    the bottom or on `G`.
  - Case-insensitive search that updates its match indices as lines arrive.
  - `stages` from `LogEvent::Stage`.
  - `source_state = Streaming | Reconnecting{attempt} | Ended{final_status?} | Gone | Failed{cause}`
  - Unit tests: overflow at 50,001 lines, follow toggling, search updating on append.
- [ ] T027 Implement the shared log viewer `crates/devtron-tui/src/views/logs.rs` (FR-014, FR-016,
  FR-017, FR-029):
  - Generic over a `LogSource` that yields `LogEvent`s. ANSI is converted with `ansi-to-tui`.
  - Keys: `f` follow, `/` search, `n`/`N` matches, `w` wrap, `G` bottom.
  - A status line for `source_state`, and "N older lines dropped" when `dropped > 0`.
  - Stage markers are shown inline.

  Depends on T011, T024 and T026.
- [ ] T028 Create `crates/devtron-tui/tests/update.rs` and `crates/devtron-tui/tests/responsiveness.rs`.
  In `update.rs`, testing pure transitions:
  - pushing and popping the view stack
  - `q` quits from a nested view and from the root; `q` typed into the filter bar is text
  - `Esc` on the root does nothing
  - a message with an old epoch is dropped and the state doesn't change
  - `r` sets `stale` and keeps `Loaded`

  In `responsiveness.rs` (FR-021, SC-004), drive `run` (T023) with a `TestBackend` and a scripted
  event stream, against a wiremock server whose responses are delayed by 5 s:
  - while the request is pending, `j`, `Esc` and `q` each change the rendered frame or end the
    loop within 100 ms of being sent
  - `q` ends the loop without waiting for the pending request

  Depends on T021 and T025.
- [ ] T029 [P] Create `crates/devtron-tui/tests/snapshots.rs`: an `insta` harness on
  `ratatui::backend::TestBackend` 100×30, with snapshots for the header (normal, expiring,
  expired), the footer, the loadable states, the too-small screen (79×24) and the log viewer
  (following, searching, dropped lines).
- [ ] T030 [P] Create `crates/devtron-tui/tests/token_never_rendered.rs`. It renders every widget and
  view state available so far with `Credentials::ApiToken` holding the canary
  `eyJ-canary-token-value`, and asserts the canary is absent from every buffer cell string, every
  `ApiError` `Display`, and the `Debug` of `Credentials` and `Session` (SC-006). Each story
  adds its views to it.

**Checkpoint**: Foundation ready. The API core is tested, the fixtures are committed, and the TUI
loop, chrome and log viewer render.

---

## Phase 3: User Story 1 - Connect and browse the application list (Priority: P1) 🎯 MVP

**Goal**: Register the first instance, verify the token, list every app with its project and
environments, and filter by name.

**Independent Test**: Launch against an instance, filter by part of a known app name, and check
the project and environments against the web UI (quickstart M1–M4).

### Tests for User Story 1

- [ ] T031 [P] [US1] Add fixture tests for A2 (`app-list.json` → `AppListResponse`, `appCount`
  equals the number of containers) and A3 (`team.json`) in `crates/devtron-api/tests/models.rs`.
- [ ] T032 [P] [US1] Add `check_roles`, `list_apps` and `teams` to
  `every_public_method_stays_in_allowlist` in `crates/devtron-api/tests/allowlist.rs`. Also add a
  paging test: wiremock returns `appCount: 150` over two pages, and `list_apps` makes exactly two
  `POST /app/list` calls with offsets 0 and 100.

### Implementation for User Story 1

- [ ] T033 [P] [US1] Create the wire models in `crates/devtron-api/src/models/apps.rs`
  (`AppListResponse { appContainers, appCount }`, `AppContainer`, `AppEnvironmentContainer`),
  `crates/devtron-api/src/models/team.rs` (`Team { id, name, active }`) and
  `crates/devtron-api/src/models/roles.rs` (`Roles`). Use Devtron's exact JSON names with
  `#[serde(rename_all = "camelCase")]`.
- [ ] T034 [US1] Add client methods in `crates/devtron-api/src/client.rs`:
  - `list_apps()` sends `POST /app/list` with
    `{offset, size: 100, sortBy: "appNameSort", sortOrder: "ASC"}` and pages until it has
    `appCount` containers (R4).
  - `teams()` sends `GET /team`.
- [ ] T035 [US1] Create the domain type in `crates/devtron-api/src/domain/apps.rs`:
  - `Application { id, name, project: Option<String>, environments: Vec<EnvRef> }`
  - `EnvRef { id, name, cluster: Option<String>, last_deployed_at: Option<Timestamp> }`
  - Skip environment entries with an empty `environmentId` or `environmentName`.
  - Resolve `projectId` against `teams()`.
  - Unit tests: empty entries are skipped, and an unknown project gives `None`.
- [ ] T036 [US1] Implement the register dialog `crates/devtron-tui/src/dialogs/register.rs` (FR-001,
  contracts/cli-config.md, First launch):
  - Fields: name, URL, credentials (the token file path, pre-filled `~/.devtron-token`; saved as
    `auth = { kind = "token_file", path }`), colour (optional). All validated with the rules of
    T019.
  - The connection is checked (A1) before saving. The first instance becomes `last_instance`.
  - Refuse a name that is already registered: "name '{name}' is already registered".
- [ ] T037 [US1] Implement the startup flow in `crates/devtron-tui/src/main.rs` and
  `crates/devtron-tui/src/app/update.rs` (FR-002–FR-005):
  - Choose the instance: `--instance`, else `last_instance`, else the register dialog.
  - An unknown `--instance` exits with code 2: "unknown instance '<name>'; registered: a, b".
  - Load the credentials (T020), then check the connection (A1).
  - `ConnectFailed{cause}` renders the contract's messages, including
    "credentials for {instance} expired on {date}", followed by the hint from `AuthConfig`
    ("replace token file {path}").
  - A permission warning banner names the file, and startup continues.
- [ ] T038 [US1] Implement the apps view `crates/devtron-tui/src/views/apps.rs` (FR-008, FR-009):
  - Columns: name, project, environments (comma-separated names).
  - `/` filter matches names case-insensitively and updates on each keystroke. `Esc` clears it.
  - Filter and selection are kept when coming back to this view (US2, scenario 4).
  - Loading, empty ("no applications visible with your access") and error states.
- [ ] T039 [US1] Add snapshots to `crates/devtron-tui/tests/snapshots.rs` (register dialog, apps
  loaded, filtered, empty, error, connect-failed expired) and to
  `crates/devtron-tui/tests/token_never_rendered.rs` (register, apps, connect-failed). Add filter
  tests to `crates/devtron-tui/tests/update.rs`.

**Checkpoint**: The MVP works. `devtron-tui` connects to an instance, lists its apps and filters
them.

---

## Phase 4: User Story 2 - See what is deployed where for one application (Priority: P2)

**Goal**: One row per environment with health, deployed tag and commit, time and author. Covers
never-deployed and deletion-pending environments.

**Independent Test**: Open a known app and compare each environment with the web UI
(quickstart M5).

### Tests for User Story 2

- [ ] T040 [P] [US2] Add fixture tests in `crates/devtron-api/tests/models.rs` for A4 (`other-env.json`),
  A5 (`detail-v2.json`), A6 (`resource-tree.json`: `status`, `nodes`, `podMetadata`) and A7 with the
  minimal `Runner` (`history.json`: `id`, `cd_workflow_id`, `workflow_type`, `status`).
- [ ] T041 [P] [US2] Add `other_env`, `app_detail` and `resource_tree` to the allowlist test in
  `crates/devtron-api/tests/allowlist.rs`. Add a wiremock test showing that an environment with
  `deploymentAppDeleteRequest: true` produces **zero** requests to `/app/detail/resource-tree`
  (R3).

### Implementation for User Story 2

- [ ] T042 [P] [US2] Create the wire models `crates/devtron-api/src/models/env.rs` (`OtherEnv`,
  `DetailV2`) and `crates/devtron-api/src/models/tree.rs` (`ResourceTree { status, nodes, podMetadata }`,
  `Node { kind, name, namespace, info: Vec<InfoItem>, createdAt, health }`, `PodMetadata { name, containers }`).
  Also create `crates/devtron-api/src/models/history.rs` with `HistoryResponse { cdWorkflows }` and
  a minimal `Runner { id, cd_workflow_id, workflow_type, status }` (snake_case, as on the wire),
  which the latest-outcome lookup needs. T049 extends it for the history view.
- [ ] T043 [US2] Add client methods in `crates/devtron-api/src/client.rs`:
  - `other_env(app_id)` (A4) and `app_detail(app_id, env_id)` (A5).
  - `resource_tree(target: TreeTarget)` (A6), where `TreeTarget::new(&EnvironmentState)` returns
    `None` when a deletion is pending or the environment was never deployed. This makes the R3
    exclusion hold by type, not by care.
  - `latest_runners(app_id, env_id)`: A7 with `offset=0&limit=3`, for the latest deployment outcome
    (R5). Add it to the allowlist test in `crates/devtron-api/tests/allowlist.rs`.
- [ ] T044 [US2] Create the domain types in `crates/devtron-api/src/domain/env.rs`:
  - `EnvironmentState`, with `deployed: Option<DeployedBuild>` (`None` when `lastDeployed` is empty,
    FR-011)
  - `DeployedBuild { image, tag (after last ':'), commit (commits[0], first 7 chars), deployed_at, deployed_by, runner_id }`
  - `Health = Healthy | Progressing | Degraded | Suspended | Missing | Hibernating | Unknown(String)`
  - `RunStatus` (data-model "Deployment", parsed case-insensitively), created here because US2
    needs it; US3 reuses it.
  - `fn latest_outcome(runners: &[Runner]) -> Option<RunStatus>`: among the newest runners, take
    the group of the first runner's `cd_workflow_id`. Return its DEPLOY runner's status, or else
    that group's newest runner's status.
  - Unit tests: tag parsing with a registry port (`host:5000/img:tag`), never deployed, an unknown
    health value, and `latest_outcome` for [POST ok, DEPLOY failed, PRE ok] of one deployment →
    `Failed`.
- [ ] T045 [US2] Implement the app detail view `crates/devtron-tui/src/views/app_detail.rs` (FR-010,
  FR-011, FR-022):
  - Rows: environment, health, last deployment (outcome), tag, commit, deployed at (relative
    time), deployed by.
  - Health (A6) and last deployment (A7, `limit=3`) are loaded per row concurrently, with at most 4
    requests in flight across the view (`buffer_unordered(4)`). A healthy row whose last deployment
    failed shows both (US2 scenario 3).
  - A 403 on one row's tree or history marks only that cell "not visible with your access", and
    the other rows render normally (FR-035).
  - Never-deployed rows show "never deployed". Deletion-pending rows show "deletion pending".
  - `r` refreshes without leaving the view. `Esc` goes back to apps with the filter kept.
  - Footer keys: `h` history, `b` builds, `p` pods. Each key shows "coming in a later story" until
    its story lands.
- [ ] T046 [US2] Add snapshots (app detail loaded, one row loading health, never deployed, deletion
  pending, healthy with a failed last deployment, one row forbidden, empty "no environments
  configured") to `crates/devtron-tui/tests/snapshots.rs`, and the
  app detail view to `crates/devtron-tui/tests/token_never_rendered.rs`.

**Checkpoint**: US1 and US2 both work on their own.

---

## Phase 5: User Story 3 - Browse deployment history (Priority: P3)

**Goal**: Deployments newest first, loaded as the user scrolls. Each entry opens its PRE/POST
stages, and each stage's log.

**Independent Test**: Open the history of an environment with known deployments and compare with
the web UI (quickstart M6).

### Tests for User Story 3

- [ ] T047 [P] [US3] Extend the A7 fixture test (`history.json`: every snake_case field of the full
  `Runner`) in `crates/devtron-api/tests/models.rs`, and add `history` and
  `cd_stage_logs` to the allowlist test in `crates/devtron-api/tests/allowlist.rs`.
- [ ] T048 [P] [US3] Write the grouping tests in `crates/devtron-api/src/domain/history.rs`
  (unit):
  - PRE+DEPLOY+POST with the same `cd_workflow_id` form one Deployment, with stages ordered PRE → POST.
  - A deployment split across two pages of 20 runners is merged, not duplicated.
  - `CANCELLED` and `Cancelled` both parse to `Cancelled`.
  - An unknown status is kept as `Unknown(text)`.

### Implementation for User Story 3

- [ ] T049 [P] [US3] Extend the wire model `crates/devtron-api/src/models/history.rs` (created
  minimal in T042). `Runner` gains `pod_status, started_on, finished_on, email_id, triggered_by,
  image, ...`, with snake_case names as on the wire. If US3 is built before US2, create the file
  here instead.
- [ ] T050 [US3] Add client methods in `crates/devtron-api/src/client.rs`:
  - `history(app_id, env_id, offset, limit = 20)` (A7), with both `filterCriteria` parameters
    (R6).
  - `cd_stage_logs(app_id, env_id, pipeline_id, wfr_id, follow)` (A8), returning a `LogStream`.
- [ ] T051 [US3] Create the domain types in `crates/devtron-api/src/domain/history.rs`, following
  data-model.md "Deployment":
  - `Deployment { id, deploy: Option<Runner>, stages, started_at, author, build, outcome }`
  - `RunStatus` parsed case-insensitively.
  - `fn merge_page(&mut Vec<Deployment>, Vec<Runner>)`
  - `fn stage_log_openable(&Runner) -> bool`, which is `pod_status` set and not `Pending`.
- [ ] T052 [US3] Implement the history view `crates/devtron-tui/src/views/history.rs` (FR-012):
  - Newest first. The next page is requested when the selection is within 5 rows of the end,
    without blocking the UI.
  - Empty state: "no deployments yet".
  - Wire the `h` key in `views/app_detail.rs`.
- [ ] T053 [US3] Implement the deployment view `crates/devtron-tui/src/views/deployment.rs` (FR-024):
  - The outcome, and the stages with their status.
  - "no pre- or post-deployment stages ran" when there are none.
  - `Enter` on an openable stage opens the shared log viewer (T027) with an A8 source.
- [ ] T054 [US3] Add snapshots (history loaded, loading more, empty, deployment with stages,
  without stages, a stage log) to `crates/devtron-tui/tests/snapshots.rs`, and both views to
  `crates/devtron-tui/tests/token_never_rendered.rs`.

**Checkpoint**: US1–US3 work on their own.

---

## Phase 6: User Story 4 - Follow CI builds and their live logs (Priority: P4)

**Goal**: A builds list per CI pipeline, plus live and finished build logs with search and
reconnection.

**Independent Test**: Start a build from the web UI and follow it in the tool until its final
status (quickstart M7, M8).

### Tests for User Story 4

- [ ] T055 [P] [US4] Add fixture tests for A9 (`ci-min.json`) and A10 (`ci-workflows.json`:
  `gitTriggers` values with **PascalCase** keys `Commit`, `Author`, `Message`) in
  `crates/devtron-api/tests/models.rs`, and add `ci_pipelines`, `builds` and `build_logs` to the
  allowlist test in `crates/devtron-api/tests/allowlist.rs`.
- [ ] T056 [P] [US4] Add a test to `crates/devtron-api/tests/sse.rs`: a 400 JSON response with
  `userMessage: "logs-not-stored-in-repository"` gives `LogEvent::Error`. The view shows it as
  "logs are not archived on this instance".

### Implementation for User Story 4

- [ ] T057 [P] [US4] Create the wire models in `crates/devtron-api/src/models/ci.rs`:
  - `CiPipelineMin { id, name, parentCiPipeline, parentAppId, pipelineType }`
  - `CiWorkflowsResponse { ciWorkflows }`
  - `CiWorkflow { id, status, podStatus, startedOn, finishedOn, triggeredByEmail, artifact, ciPipelineId, gitTriggers: HashMap<String, GitTrigger> }`
  - `GitTrigger` uses `#[serde(rename_all = "PascalCase")]`.
- [ ] T058 [US4] Add client methods in `crates/devtron-api/src/client.rs`:
  - `ci_pipelines(app_id)` (A9).
  - `builds(pipeline_id, offset, size = 20)` (A10). Both parameters are always sent.
  - `build_logs(pipeline_id, workflow_id, follow)` (A11), returning a `LogStream`. `follow` is
    `false` for finished builds.
- [ ] T059 [US4] Create the domain types in `crates/devtron-api/src/domain/ci.rs`:
  - `CiPipeline { id, name, parent_pipeline_id, parent_app_id }`
  - `fn builds_source(&CiPipeline) -> pipeline id`: the parent pipeline when
    `parentCiPipeline > 0` (R7).
  - `Build { id, pipeline_id, status: RunStatus, started_at, finished_at (all-zero date → None), author, commit (short), message (first line), artifact, pod_status }`
- [ ] T060 [US4] Implement the builds view `crates/devtron-tui/src/views/builds.rs` (FR-013):
  - Newest first, loading more on scroll.
  - A pipeline selector (`Tab`) when the app has more than one CI pipeline.
  - Empty state: "no builds yet".
  - `Enter` opens the shared log viewer with an A11 source. The final status appears when `End`
    arrives (FR-015). A reconnection shows "reconnecting (n/3)…" (US4, scenario 5).
  - Wire the `b` key in `views/app_detail.rs`.
- [ ] T061 [US4] Add snapshots (builds with one pipeline, several pipelines, empty, a live build
  log, a finished build log, logs not archived) to `crates/devtron-tui/tests/snapshots.rs`, and the
  builds view to `crates/devtron-tui/tests/token_never_rendered.rs`.

**Checkpoint**: US1–US4 work on their own.

---

## Phase 7: User Story 5 - Follow the runtime logs of an application's pods (Priority: P5)

**Goal**: A pod list per environment, then live container logs with container switching,
pre-restart logs, and detection of a pod that is gone.

**Independent Test**: Open an environment's pods, open one, and see fresh lines arrive (quickstart
M9).

### Tests for User Story 5

- [ ] T062 [P] [US5] Write the pod extraction tests in `crates/devtron-api/src/domain/pods.rs`
  (unit) against `resource-tree.json`:
  - only `kind == "Pod"` nodes are kept
  - `Restart Count` absent gives `restarts == 0` (R9), and `"3"` gives 3
  - containers come from `podMetadata` matched by name
- [ ] T063 [P] [US5] Add `pod_logs` to the allowlist test in
  `crates/devtron-api/tests/allowlist.rs`. Add a wiremock test that checks the exact query:
  - `containerName`, `follow=true`, `previous`
  - `appId={clusterId}|{appId}|{envId}`, `appType=0`
  - `deploymentType` 0 for `helm` and 1 for `argo_cd`
  - `namespace`, `tailLines=500`

### Implementation for User Story 5

- [ ] T064 [US5] Create the domain types in `crates/devtron-api/src/domain/pods.rs`:
  - `Pod { name, namespace, status (info "Status Reason"), ready (info "Containers"), restarts ("Restart Count", absent → 0), created_at, containers }`
  - `fn pods(&ResourceTree) -> Vec<Pod>` (data-model Pod/Container). Init and ephemeral
    containers are left out.
- [ ] T065 [US5] Add the client method `pod_logs(PodLogTarget { cluster_id, app_id, env_id, namespace, pod, container, deployment_type, previous })`
  (A12) in `crates/devtron-api/src/client.rs`, returning a `LogStream`. A non-200 status is fatal
  even if bytes follow (R8).
- [ ] T066 [US5] Implement the pods view `crates/devtron-tui/src/views/pods.rs` (FR-025):
  - Columns: name, status, ready, restarts, age.
  - Empty state: "no running pods".
  - `Enter` opens the first or only container. With several containers, the container picker
    dialog `crates/devtron-tui/src/dialogs/container_picker.rs` appears first.
  - A 403 on the tree marks the view "not visible with your access". A 403 on one pod's log stream
    only affects that log viewer (FR-035).
  - Load `app_detail` (A5) for the namespace and `deploymentAppType`.
  - Wire the `p` key in `views/app_detail.rs`.
- [ ] T067 [US5] Extend the log viewer for pod sources in `crates/devtron-tui/src/views/logs.rs`
  (FR-026–FR-028):
  - `c` switches container without leaving the viewer.
  - `P` reopens the stream with `previous=true`.
  - On `LogEvent::Closed`, re-fetch the resource tree. If the pod is missing →
    `source_state = Gone` with "pod {name} is gone", keeping the lines. Otherwise reconnect with
    `Last-Event-ID`.
- [ ] T068 [US5] Add snapshots (pods loaded, empty, container picker, pod log following, pod gone)
  to `crates/devtron-tui/tests/snapshots.rs`, and the pods view to
  `crates/devtron-tui/tests/token_never_rendered.rs`.

**Checkpoint**: US1–US5 work on their own.

---

## Phase 8: User Story 6 - Register and switch between Devtron instances (Priority: P6)

**Goal**: An instance list with token status. Register, remove and switch, with no data leaking
from the previous instance.

**Independent Test**: Register two instances, switch back and forth, and check that the header,
colour and app list change with nothing left over (quickstart M10, M11).

### Tests for User Story 6

- [ ] T069 [P] [US6] Write `crates/devtron-tui/tests/switch_race.rs` (FR-033, SC-011), using two
  wiremock servers A and B:
  1. A's `/app/list` responds after a 2 s delay. The test switches to B during the delay, and A's
     apps never appear in `State`.
  2. While following an A pod log that streams one line every 100 ms, switch to B. Afterwards no
     A line is appended, and A's stream task has ended (cancellation observed).
  3. Credentials stay with their instance: every request B receives carries B's token file
     content, A never receives B's token, and B never receives A's token, including requests sent
     around the switch.
- [ ] T070 [P] [US6] Add registry tests to `crates/devtron-tui/tests/update.rs`:
  - a duplicate name is refused
  - removal asks for confirmation (default No) and leaves the token file untouched (FR-031)
  - a missing `last_instance` at launch opens the instance list instead of exiting (FR-034)

### Implementation for User Story 6

- [ ] T071 [US6] Implement the instances view `crates/devtron-tui/src/views/instances.rs` (FR-030,
  FR-032):
  - Columns: name (in its colour), URL, credential status (`valid until {date}` / `expired` /
    `unreadable`), computed from the instance's `auth` without network access (for a token file:
    readable, then `exp`).
  - `a` opens the register dialog (T036). `d` removes after the confirm dialog, default **No**.
    `Enter` switches.
  - Reachable through `:instances` and `:ctx`.
- [ ] T072 [US6] Implement switching in `crates/devtron-tui/src/app/update.rs` and
  `crates/devtron-tui/src/session.rs` (FR-003, FR-033):
  - Cancel the old session and increment the epoch.
  - Clear every view's state and reset the stack to the new instance's apps view.
  - Load the target's credentials from its own `auth` (T020), then check the connection (A1).
  - Save `last_instance` after a successful connection.
  - Expired credentials on the target show the US1 expired message (US6, scenario 6).
- [ ] T073 [US6] Implement the fallback at launch in `crates/devtron-tui/src/main.rs` (FR-034): when
  the last-used instance is gone from the registry or unreachable, open the instances view with
  the cause shown, instead of exiting.
- [ ] T074 [US6] Add snapshots (instances list with valid, expired and unreadable tokens, the remove
  confirmation, the header after switching to a red instance) to
  `crates/devtron-tui/tests/snapshots.rs`, and the instances view to
  `crates/devtron-tui/tests/token_never_rendered.rs`.

**Checkpoint**: All six stories work on their own.

---

## Phase 9: Polish & Cross-Cutting Concerns

- [ ] T075 Finish `crates/devtron-api/tests/allowlist.rs`: assert that
  `every_public_method_stays_in_allowlist` exercises all 12 routes A1–A12, so a route with no
  caller or a caller with no route fails (SC-005).
- [ ] T076 [P] Add a performance check in `crates/devtron-tui/tests/perf.rs` (SC-001). It renders
  the apps view with 200 synthetic apps and applies a 3-character filter. The re-render must finish
  in under 16 ms in `--release`. Mark the test `#[ignore]` and run it in CI with
  `cargo test --release -- --ignored perf`.
- [ ] T077 [P] Write `README.md` covering what it is, install (`cargo install --git`), the config
  example from contracts/cli-config.md with `devtron.example.com`, the keys (link to
  contracts/keybindings.md), running the tests, and the read-only guarantee. No instance-specific
  names (CLAUDE.md).
- [ ] T078 Run a final leak scan: `git grep -n -iE -f fixtures-raw/denylist.txt` over tracked files
  must print nothing, and `cargo test -p devtron-api --test fixtures_are_scrubbed` passes.
- [ ] T079 Run the manual checks M1–M13 of `specs/001-readonly-browsing/quickstart.md` against the
  reference instance, and record the results (pass or fail per row, no instance names) at the end
  of `specs/001-readonly-browsing/checklists/requirements.md`.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: none. T001 comes first. T002–T005 can run in parallel after it.
- **Foundational (Phase 2)**: depends on Setup and blocks every story. The internal chains are:
  - T006–T009 → T010 → T017
  - T012 → T013 → T018
  - T011 → T016, T027
  - T019–T021 → T022, T023
  - T021 + T024 → T025 → T028
  - T024 + T026 + T011 → T027
- **User stories (Phases 3–8)**: each depends on Foundational only. After it, they can proceed in
  any order or in parallel.
  - US2's detail view is reached from US1's list. It can be tested alone through
    `update.rs`/snapshots with a pre-set state.
  - US3, US4 and US5 each wire one key in `views/app_detail.rs` (T052, T060, T066). If US2 isn't
    done, that one-line wiring waits, and the view is still testable directly.
  - US6 reuses US1's register dialog (T036).
  - US2 creates the minimal A7 wire model and `RunStatus` (T042, T044) for the latest-deployment
    column. US3 extends them (T049, T051). Whichever story comes first creates the file.
- **Polish (Phase 9)**: after the stories it covers. T075 needs all of US1–US5.

### Within each story

Fixture and allowlist tests first, and they fail until the wire model and client method exist.
Then wire model → client method → domain → view → snapshots.

### Parallel Opportunities

- Setup: T002, T003, T004, T005.
- Foundational: T006, T007, T008 together; T011, T012, T019, T020, T024, T026 together; then T014,
  T015, T016, T029, T030 together.
- In each story, the `[P]` test tasks and the `[P]` wire-model task touch different files.
- Across stories: once Foundational is done, US3, US4 and US5 touch disjoint model, client-method,
  domain and view files. The exceptions are the shared files `client.rs`, `models.rs`,
  `allowlist.rs` and `snapshots.rs`, so edits to those need to be serialised or merged with care.

---

## Parallel Example: User Story 4

```bash
# Tests and wire model at once (different files):
Task: "T055 Fixture tests for A9/A10 in crates/devtron-api/tests/models.rs + allowlist entries"
Task: "T056 logs-not-stored test in crates/devtron-api/tests/sse.rs"
Task: "T057 Wire models in crates/devtron-api/src/models/ci.rs"
# Then, in order:
Task: "T058 Client methods ci_pipelines/builds/build_logs"
Task: "T059 Domain CiPipeline/Build in crates/devtron-api/src/domain/ci.rs"
Task: "T060 Builds view in crates/devtron-tui/src/views/builds.rs"
```

---

## Implementation Strategy

### MVP first (User Story 1 only)

1. Phase 1 Setup, then Phase 2 Foundational.
2. Phase 3 (US1).
3. **Stop and validate** with quickstart M1–M4 on the reference instance. The tool already answers
   "which apps exist and where are they deployed".

### Incremental delivery

US2 (what is deployed where), then US3 (history), US4 (builds and logs), US5 (pod logs) and US6
(several instances). Each checkpoint is a working tool. Commit and push at every checkpoint.

### Suggested cut if time is short

US1 + US2 + US5 covers the incident workflow (SC-009) without history or builds, since US5 only
depends on the shared log viewer from Foundational.

---

## Notes

- `[P]` means a different file with no dependency on an incomplete task.
- Commit after each task or logical group with a one-line subject, no trailer (CLAUDE.md).
- Never add a fixture without running `fixtures_are_scrubbed`. Never paste real log lines into a
  test.
