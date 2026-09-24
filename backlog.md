# Backlog

Features agreed as out of scope for now. Each one becomes a `/speckit-specify` when it is picked
up. Delete an entry once its spec exists under `specs/`.

| Feature | Origin | Notes |
|---|---|---|
| Helm apps (chart store) | 001, Q1 | List and detail for Helm apps deployed through the chart store. They have no CI builds. The detail view shows release, chart version and resource status instead. |
| Devtron jobs | 001, Q1 | Job list and job runs with logs. Jobs have runs but no environments, so this needs its own view shape. |
| Mutating actions | Constitution I | Trigger build, deploy, rollback, rotate pods. Each one needs a confirmation dialog, must be disabled by `--readonly`, and must have a test proving both. |
| Headless commands | 001, Assumptions | `clap` subcommands with `-o json` for the views, for scripts and CI (constitution IV). |