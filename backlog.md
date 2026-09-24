# Backlog

Features agreed as out of scope for now. Each one becomes a `/speckit-specify` when it is picked
up. Delete an entry once its spec exists under `specs/`.

| Feature | Origin | Notes |
|---|---|---|
| Helm apps (chart store) | 001, Q1 | List and detail for Helm apps deployed through the chart store. They have no CI builds. The detail view shows release, chart version and resource status instead. |
| Devtron jobs | 001, Q1 | Job list and job runs with logs. Jobs have runs but no environments, so this needs its own view shape. |
| Mutating actions | Constitution I | Trigger build, deploy, rollback, rotate pods. Each one needs a confirmation dialog, must be disabled by `--readonly`, and must have a test proving both. |
| Headless commands | 001, Assumptions | `clap` subcommands with `-o json` for the views, for scripts and CI (constitution IV). If scripts need credentials from the environment, use per-instance variables such as `DEVTRON_TOKEN_<NAME>` (constitution II), never a single global one. |
| Keyring credentials | 001, analysis I1 | New `auth` kind that stores the token in the OS keyring. WSL2 usually has no Secret Service, so it needs a documented fallback to `token_file`. Requires its own spec (constitution II). |
| Username/password login | 001, analysis I1/G1 | New `auth` kind that logs in through Devtron's session endpoint and holds the resulting session cookie as `Credentials::SessionCookie`. Needs rules for storing or re-prompting the password (constitution II). |
| Record a real A8 fixture | 001, plan Complexity Tracking | Replace the synthesized CD stage-log fixture with a recording once an instance has pre/post deployment stages (constitution V). |
