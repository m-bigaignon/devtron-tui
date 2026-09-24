# Research: Read-Only Browsing of a Devtron Instance

**Feature**: 001-readonly-browsing | **Date**: 2026-09-24

Sources:

- **Source code**:
  - `devtron-labs/devtron` at `0874dcaf`, the commit the reference instance reports
    through `/orchestrator/version` (built 2026-07-06).
  - `devtron-labs/dashboard` (the web UI).
  - `devtron-labs/devtron-fe-common-lib`.
  - `devtron-labs/kubelink` (the Helm backend).
- **Live probes**: read-only calls against the maintainer's reference instance on 2026-09-24. Its
  address is only in the local config. The raw responses are
  kept in the local, git-ignored `fixtures-raw/` and are not committed.

In the Devtron repos, "UI" means `dashboard` and "backend" means `devtron`. Every path is under
`/orchestrator`.

## R1. Authentication and token validation

- **Decision**:
  - Send the token as the `token: <jwt>` header.
  - Validate at startup and on each instance switch with `GET /user/check/roles`. It returns
    `{roles: [string], superAdmin: bool}`.
  - Decode the JWT payload locally, without checking its signature, to read `exp` and `email`.
  - Classify failures by HTTP status:
    - connection error → unreachable
    - 401 → invalid token, or expired if the local `exp` is in the past
    - 403 → missing permission
- **Rationale**:
  - The backend only reads `token`. `Authorization: Bearer` is ignored
    (`authenticator/middleware/AuthMiddleware.go:29-51`).
  - On the reference instance, a wrong token gets `401 {"code":401,"result":"Unauthorized"}` (probed),
    with no `errors` array.
  - The token is **not super-admin** (`superAdmin: false`, 27 roles). Views must therefore handle
    partial visibility: FR-020 errors, and the spec's "lacks permission" edge case.
- **Alternatives considered**:
  - `GET /version` also sits behind auth, but it says nothing about roles.
  - Checking the JWT signature is impossible without the server secret, and not needed.

## R2. Response envelope and errors

- **Decision**:
  - Parse every JSON response as `{code, status?, result?, errors?}`.
  - An error is any HTTP status ≥ 400. The message is taken from `errors[0].userMessage` (a string
    or an object, displayed as text), falling back to `result` when it is a string, then to the
    status text.
- **Rationale**:
  - All four fields are `omitempty` (`api/restHandler/common/apiError.go:145-150`).
  - Many RBAC failures send `{"code":403,"status":"Forbidden","result":"unauthorized user"}` with
    no `errors` (`AppListingRestHandler.go:701-703`).
  - An error that happens before a stream starts comes back as a JSON envelope, not SSE.
- **Alternatives considered**: deciding success from whether `errors` is present. Rejected,
  because it treats 403s as successes.

## R3. Enforcing read-only (FR-007, SC-005, constitution I)

- **Decision**:
  - The API crate exposes one typed method per endpoint and no public generic request function.
  - Every request goes through one private function that checks an explicit **allowlist** of
    (method, path pattern) pairs, the "Contract" column in `contracts/devtron-api.md`. It refuses
    anything else before any bytes are sent.
  - A unit test runs every public method against a mock server and asserts that each request
    matches the allowlist. A second test asserts that the allowlist contains no method other than
    `GET` and the one `POST /app/list`.
- **Rationale**:
  - The HTTP method is not a reliable signal: `POST /app/list` is a pure query
    (`AppListingRestHandler.go:304-437`).
  - An allowlist makes "read-only" checkable in code rather than a matter of review.
- **Known server-side effects of allowlisted reads**, recorded so the "read-only" promise is
  accurate:
  - Every call records the token's last-used time (`UserService.go:1248-1250`).
  - `GET /app/detail/resource-tree` updates `app_status` in the background, which the web UI also
    triggers every few seconds. If the pipeline already has `deploymentAppDeleteRequest = true` and
    the Argo app is gone, it also marks the pipeline deleted (`AppListingRestHandler.go:614-632`).
    **The tool therefore never requests the resource tree for an environment whose
    `deploymentAppDeleteRequest` is `true`**, and shows it as "deletion pending" instead.
- **Alternatives considered**:
  - Blocking every non-GET method would break the app list.
  - Relying on the token's roles is not enough: the token has write roles.

## R4. Application list (US1)

- **Decision**:
  - Use `POST /app/list` with `{offset, size: 100, sortBy: "appNameSort", sortOrder: "ASC"}`, and
    fetch more pages until `appContainers` totals `appCount`.
  - Resolve project names with `GET /team` (`[{id, name, active}]`).
  - List each app's environments from `appContainers[].environments[]`. Entries without
    `environmentId` or `environmentName` are skipped, as the UI does.
  - Filter on the client.
- **Rationale**:
  - The SQL returns Devtron apps only (`app_type = 0`), which matches Q1.
  - On the reference instance, the list's per-environment `teamName`, `appStatus`,
    `lastDeployedImage` and `lastDeployedBy` are empty for all 112 entries, while
    `environmentName`, `clusterName` and `lastDeployedTime` are filled (probed). So the list can't
    show health. That is consistent with FR-008, which asks only for name, project and
    environments.
  - 37 apps fit in one page, and filtering on the client makes every keystroke instant (FR-009).
- **Alternatives considered**:
  - The server-side `appNameSearch` needs a request per keystroke, which gets no benefit at this
    size.
  - `/app/autocomplete` has no environments.

## R5. Application detail (US2)

- **Decision**:
  - Fetch the environment rows with `GET /app/other-env?app-id=`. It provides `environmentId`,
    `environmentName`, `clusterId`, `pipelineId`, `lastDeployed`, `lastDeployedBy` (email),
    `lastDeployedImage`, `commits[]`, `latestCdWorkflowRunnerId`, `prod` and
    `deploymentAppDeleteRequest`.
  - The deployed build is the image tag (text after the last `:` of `lastDeployedImage`) and the
    short `commits[0]`.
  - Health comes from each environment's `GET /app/detail/resource-tree?app-id=&env-id=` →
    `result.status`, fetched concurrently (at most 4 at a time).
  - The tree is skipped when `lastDeployed` is empty ("never deployed", FR-011) or when a deletion
    is pending (R3).
  - `GET /app/detail/v2` is fetched only when a view needs `namespace` and `deploymentAppType`
    (pods, R9).
- **Rationale**:
  - `other-env` is what the UI Overview uses. Its `appStatus` is `""` on the reference instance
    (probed), so health has to come from the resource tree (`status: "Healthy"`, probed).
  - The UI only fetches the tree when the pipeline has been triggered (`service.ts:137-142`).
- **Alternatives considered**: `GET /app/detail/v2` per environment. It has no health and no
  deploying user, so it adds a call without adding the fields we need.

## R6. Deployment history and pre/post stages (US3)

- **Decision**:
  - Use `GET /resource/history/deployment/cd-pipeline/v1?filterCriteria=application/devtron-application|id|{appId}&filterCriteria=environment|id|{envId}&offset=&limit=20`.
  - Its rows are *runners* (`workflow_type` PRE/DEPLOY/POST, snake_case fields). The tool groups
    them into deployments by `cd_workflow_id`.
  - Paging counts runners. A deployment split across a page boundary is merged when the next page
    arrives.
  - A deployment's outcome is the status of its DEPLOY runner. Its stages are its PRE and POST
    runners.
  - Stage logs come from `GET /app/cd-pipeline/workflow/logs/{appId}/{envId}/{pipelineId}/{wfrId}`
    (SSE, R8). The stream is only opened when `pod_status` is set and not `Pending`, as in the UI.
- **Rationale**:
  - This is the endpoint the current UI uses (`CICDHistory/service.tsx:290-308`), and it responds
    on the reference instance (19 rows, probed). The legacy
    `/app/cd-pipeline/workflow/history/{appId}/{envId}/{pipelineId}` also responds (probed), but the
    UI no longer uses it.
  - **No PRE/POST runner exists** in the last 40 runners of 40 deployed environments on
    the reference instance. The stage-log fixture is therefore built from the CI log capture. Both come
    from the same SSE writer (`PipelineConfigRestHandler.go:484-587`), and the fixture is marked as
    synthesized. The FR-024 "no stages ran" state is what real data shows today.
- **Alternatives considered**: using the legacy endpoint as the primary source. Rejected: it
  needs `pipelineId` and is being phased out of the UI. It is not implemented as a fallback
  either, since both exist at `0874dcaf`.

## R7. CI builds (US4)

- **Decision**:
  - List the app's CI pipelines with `GET /app/{appId}/ci-pipeline/min`. If there are several,
    the build view shows a pipeline selector.
  - A linked CI pipeline (`parentCiPipeline > 0`) reads its builds from the parent pipeline.
  - List builds with `GET /app/ci-pipeline/{pipelineId}/workflows?offset=&size=20`. Both
    parameters are required.
  - The commit and author come from `gitTriggers[*]`, whose keys are **PascalCase**: `Commit`,
    `Author`, `Message`, ….
- **Rationale**:
  - A sampled app on the reference instance has 3 CI pipelines (probed), so a merged list would need
    per-pipeline paging to be interleaved.
  - PascalCase keys confirmed by probe and by `CiWorkflowRepository.go:189-200`.
- **Alternatives considered**: merging all pipelines into one list sorted by time. Rejected
  because it complicates paging and hides which pipeline a build belongs to.

## R8. Log streaming (US3, US4, US5, FR-014/016/017/029)

- **Decision**:
  - All three log kinds are Server-Sent Events.
  - One `LogStream` in the API crate turns an SSE response into `LogEvent`s: `Line`, `Stage`,
    `End`, `Error`, `Reconnecting`, `Gone`.
  - The parser is `eventsource-stream` over `reqwest::Response::bytes_stream()`.
  - Reconnection sends `Last-Event-ID` with the last `id` received: up to 3 attempts with backoff,
    as the UI does.
  - ANSI colour codes are converted for display (`ansi-to-tui`).
  - The log buffer is a ring of at most 50,000 lines, with a counter of dropped lines.
  - Specific frames:
    - **CI and CD-stage logs**: `data:START_OF_STREAM`, `event:START_OF_STREAM`, per-line
      `id:<n>\ndata:<line>`, `data:END_OF_STREAM` + `event:END_OF_STREAM`, `UNEXPECTED_END_OF_STREAM`
      and `event:RECONNECT_STREAM`. Lines starting `STAGE_INFO|{json}` become stage markers.
      Probed on a finished build: 1,683 lines, 12 `STAGE_INFO`, clean START and END.
      `followLogs=false` is used for finished builds.
    - **Pod logs**: `id: <unixNanos>` (with a space), `data:<text>`, `event:PING` every 30 s,
      `event:CUSTOM_ERR_STREAM`, and **no END frame**: the connection just closes. Probed with
      `tailLines=5`, which returned 5 lines.
      - When the connection closes, the tool re-checks the resource tree to tell "pod gone"
        (FR-028) from a network drop, which is reconnected.
      - A non-200 status is fatal, because the handler can write a 403 body and then stream anyway
        (`k8sApplicationRestHandler.go:559-567`).
- **Rationale**: the backend picks live or archived (blob) logs by itself
  (`HandlerService.go:1819-1852`), so one code path serves running and finished builds. On
  the reference instance, a finished build's logs came back as a full SSE stream (probed).
- **Alternatives considered**:
  - `reqwest-eventsource` 0.6 was last released in March 2024 and is pinned to an older reqwest.
  - `GET .../logs/old` is unused by the UI and its format is unconfirmed.
  - A buffer with no limit breaks FR-017.

## R9. Pods and pod logs (US5)

- **Decision**:
  - Pods come from the resource tree's `nodes[]` with `kind == "Pod"`:
    - status: info item `Status Reason`
    - readiness: `Containers` (e.g. `1/1`)
    - restarts: `Restart Count`, **absent means 0**
    - age: `createdAt`
  - Container names come from `podMetadata[]` (matched by pod name).
  - Logs come from `GET /k8s/pods/logs/{pod}?containerName=&follow=true&previous=&appId={clusterId}|{appId}|{envId}&appType=0&deploymentType=&namespace=&tailLines=500`.
  - `deploymentType` is `0` for `helm` and `1` for `argo_cd`, from `detail/v2.deploymentAppType`.
  - `previous=true` gives the pre-restart logs (FR-027).
- **Rationale**:
  - On the reference instance, deployments are **Helm** (`deploymentAppType: "helm"`), and the pod info
    items are `Containers`, `Node` and `Status Reason` only (probed).
  - kubelink only adds `Restart Count` when restarts > 0 (`kubelink` pod info builder,
    `if restarts > 0`).
  - The route only matches if `containerName` and `follow` are present
    (`k8sApplicationRouter.go:61-64`).
  - 500 tail lines is the UI default.
- **Alternatives considered**: reading restart counts from the pod manifest through the k8s
  resource API. That is an extra POST endpoint for a value the tree already implies.

## R10. Stack and crates (constitution: Technology & Constraints)

| Crate | Version | Role | In constitution list? |
|---|---|---|---|
| `ratatui` | 0.30 (MSRV 1.88) | TUI | yes |
| `crossterm` | 0.29 | terminal backend and events (`event-stream` feature) | yes |
| `tokio` | 1.53 | runtime | yes |
| `reqwest` | 0.13 (`rustls`, `json`, `stream`) | HTTP | yes |
| `serde`, `serde_json` | 1.0 | models | yes |
| `clap` | 4.6 | `--instance`, `--readonly`, `--config` | yes |
| `color-eyre` | 0.6 | errors, panic hook | yes |
| `eventsource-stream` | 0.2 | SSE parsing (R8) | **no, justified in plan** |
| `ansi-to-tui` | 8.0 (on `ratatui-core` 0.1) | ANSI log colours | **no, justified in plan** |
| `tokio-util` | 0.7 | `CancellationToken` for switching (R11) | **no, justified in plan** |
| `futures` | 0.3 | stream combinators | **no, justified in plan** |
| `secrecy` | 0.10 | token type that redacts itself (constitution II) | **no, justified in plan** |
| `base64` | 0.23 | decoding the JWT payload for `exp` (R1) | **no, justified in plan** |
| `toml` | 1.1 | config file | **no, justified in plan** |
| `directories` | 6.0 | XDG config path | **no, justified in plan** |
| `jiff` | 0.2 | timestamps, "age" and "3h ago" formatting | **no, justified in plan** |
| `wiremock` (dev) | 0.6 | HTTP mocks for API and switch tests | dev-only |
| `insta` (dev) | 1.48 | view snapshots with `ratatui::backend::TestBackend` | dev-only |

The Rust 1.90 toolchain meets every MSRV (the highest is ratatui's 1.88).

## R11. Architecture: event loop and switching instances (FR-021, FR-033, SC-004, SC-011)

- **Decision**:
  - A message loop in the style of the Elm architecture. `update(&mut State, Msg) -> Vec<Cmd>` is
    pure and synchronous. A `Cmd` runs on tokio and sends back `Msg`s over an `mpsc` channel. The
    render loop draws `State` and never waits on I/O.
  - Each active instance is a `Session { instance, client, cancel: CancellationToken, epoch: u64 }`.
  - Switching cancels the old token (which aborts HTTP requests and SSE streams), increments
    `epoch`, and replaces all view state.
  - Every `Msg` coming from a task carries its `epoch`. `update` drops messages from an old epoch.
- **Rationale**: cancelling alone is not enough. A response already in the channel when the
  switch happens would still arrive. Tagging messages with the epoch closes that race, and a
  test can check it deterministically (SC-011).
- **Alternatives considered**: shared `Arc<Mutex<State>>` mutated by tasks. Rejected: it can't be
  tested without timing tricks, and it invites the render loop to block on locks.

## R12. Fixtures and scrubbing (constitution II and V)

- **Decision**:
  - A dev-only example binary records responses for the allowlisted endpoints from a configured
    instance into `fixtures-raw/` (git-ignored).
  - A scrubber rewrites them into `crates/devtron-api/tests/fixtures/`:
    - emails → `user<n>@example.com`
    - the instance host, and every domain in the local deny-list, → `devtron.example.com`
    - image registries → `registry.example.com`
    - git remotes → `https://git.example.com/...`
    - commit messages → `commit message <n>`
    - JWT-looking strings → dropped
    - log lines → replaced with `log line <n>`, keeping the SSE framing and `STAGE_INFO` markers
  - A test (`fixtures_are_scrubbed`) fails if any committed fixture matches a deny pattern:
    - generic committed patterns: any email outside `example.com`, any JWT-like string, any
      hostname outside `*.example.com`;
    - local patterns: from the git-ignored `fixtures-raw/denylist.txt`, which holds the real
      organisation and domain names. When that file is absent (CI, other clones), only the
      generic patterns run.
- **Rationale**: this repository is public (GitHub). An unscrubbed recording would leak internal
  hostnames, user emails and possibly secrets printed in build logs. The deny-list test makes
  scrubbing mandatory rather than something to remember. The organisation-specific patterns stay
  local, because committing them would publish the names they protect. The same rule applies to
  specs and plans: they name "the reference instance", never its address.
- **Alternatives considered**:
  - Hand-writing fixtures from the specs would defeat constitution V's purpose.
  - Committing the raw recordings is forbidden by constitution II.

## R13. Configuration

- **Decision**:
  - The config file is `~/.config/devtron-tui/config.toml` (from `directories`):
    `last_instance = "<name>"` plus `[instances.<name>]` with `url`, `token_file` (a leading `~` is
    expanded) and an optional `color`.
  - Writes are atomic (temp file, then rename).
  - Tokens resolve as follows: `DEVTRON_TOKEN` for the active instance only, then its
    `token_file`, then `~/.devtron-token`.
  - The permission warning fires when `mode & 0o077 != 0`.
  - `--readonly` exists from this feature on and is effectively always true, since there are no
    mutating actions yet. It is wired in so later features get it without changing the CLI.
- **Rationale**: the file the user already has works as-is:

  ```toml
  last_instance = "work"

  [instances.work]
  url = "https://devtron.example.com"
  token_file = "~/.devtron-token"
  ```

- **Alternatives considered**:
  - An OS keyring: WSL2 has no reliable secret service, and constitution II requires token files.
  - YAML: `toml` is the Rust convention for configs you edit by hand.
