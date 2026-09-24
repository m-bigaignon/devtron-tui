# Quickstart: Validating Read-Only Browsing

How to prove that feature 001 works end to end. The endpoint behaviour is described in
[contracts/devtron-api.md](contracts/devtron-api.md), and the keys in
[contracts/keybindings.md](contracts/keybindings.md).

## Prerequisites

- Rust 1.90 or later. The sections below assume `cargo` is on `PATH`.
- A Devtron instance and an API token with read access, for the manual checks only. Automated
  tests need neither.
- For recording fixtures: a git-ignored `fixtures-raw/denylist.txt` with one pattern per line
  (organisation and domain names to keep out of the repo).

## 1. Automated gates (no network)

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
```

Expected: all pass. The tests that carry the spec's hard guarantees:

| Test | Proves |
|---|---|
| `devtron-api/tests/allowlist.rs` | every request is in the allowlist, and `POST` only goes to `/app/list` (FR-007, SC-005) |
| `devtron-api/tests/models.rs` | every endpoint's scrubbed fixture deserializes (constitution V) |
| `devtron-api/tests/errors.rs` | unreachable, invalid, expired and forbidden are told apart (FR-003, SC-008) |
| `devtron-api/tests/sse.rs` | START/END framing, `STAGE_INFO`, reconnect with `Last-Event-ID`, pod stream close, a streamed line delivered within 100 ms locally (FR-014, US4 scenario 5, FR-028, SC-003) |
| `devtron-api/tests/fixtures_are_scrubbed.rs` | no email, host or JWT that should be private is committed (constitution II) |
| `devtron-tui/tests/responsiveness.rs` | `Esc`, `q` and navigation take effect within 100 ms while a request is stalled (FR-021, SC-004) |
| `devtron-tui/tests/switch_race.rs` | a response or log line from the old instance never shows after a switch (FR-033, SC-011) |
| `devtron-tui/tests/token_never_rendered.rs` | the token never appears in any rendered frame or error (FR-006, SC-006) |
| `devtron-tui/tests/snapshots.rs` | every view renders its loading, empty, error and "too small" states (FR-020) |

## 2. Recording or refreshing fixtures (optional, needs an instance)

```bash
cargo run -p devtron-api --example record -- --instance <name>
cargo test -p devtron-api --test fixtures_are_scrubbed
```

Expected: raw responses land in `fixtures-raw/` (git-ignored), and scrubbed copies land in
`crates/devtron-api/tests/fixtures/`. The scrub test passes. Check `git diff` before committing.

## 3. Manual checks against a real instance

Run `cargo run -p devtron-tui`. Each row is a user story's independent test.

| # | Story | Steps | Expected |
|---|---|---|---|
| M1 | US1 | Move the config aside, then launch | Register form. After saving, the app list loads within 3 s, and the header shows the instance name, email and token expiry. |
| M2 | US1 | `/`, type part of an app name | The list narrows on each keystroke. `Esc` restores it. |
| M3 | US1 | `chmod 644` the token file, then launch | A warning names the file, and the tool continues. Restore with `chmod 600`. |
| M4 | US1 | Point the instance's `auth.path` to a file holding an invalid token | "credentials for … were rejected", not a crash. `:instances` is still usable. |
| M5 | US2 | `Enter` on an app | One row per environment with health, tag, commit, time and author. Never-deployed environments are marked. Matches the web UI. |
| M6 | US3 | On an environment, `h` | Deployments newest first. Scrolling loads older ones. `Enter` shows stages, or says that no stages ran. |
| M7 | US4 | Start a build from the web UI, then `b` and `Enter` on it | Lines stream live, `f` toggles follow, `/` then `n` jumps between matches, and the final status shows at the end. |
| M8 | US4 | During M7, disconnect the network briefly | "reconnecting…", then the stream resumes without duplicate or missing lines. |
| M9 | US5 | On an environment, `p` and `Enter` | Live pod logs. `c` switches container. `P` shows the previous logs of a restarted container. |
| M10 | US6 | `:instances`, `a` to register a second instance, `Enter` on it | The header name and colour change, and the app list is the new instance's, within 5 s. |
| M11 | US6 | Follow pod logs (M9), then switch instance | The stream stops, and no line from the old instance appears afterwards. |
| M12 | edge | Resize the terminal to 70×20 | "terminal too small". Back at 80×24 or larger, the view is restored. |
| M13 | edge | In three separate runs: press `Ctrl-C`; run `kill <pid>` (SIGTERM) from another shell; run `kill -HUP <pid>` | In each case the shell is usable without running `reset`. SIGKILL is out of scope (SC-007). |

## 4. Read-only audit (SC-005)

Throughout the manual checks, the Devtron audit log for this token shows only the calls in
[contracts/devtron-api.md](contracts/devtron-api.md), and nothing on the instance changes.
