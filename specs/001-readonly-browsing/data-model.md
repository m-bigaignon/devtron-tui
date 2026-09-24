# Data Model: Read-Only Browsing of a Devtron Instance

These are the domain types the TUI works with. Each one is built from one or more wire models in
`devtron-api` (`contracts/devtron-api.md`).

Wire models:

- mirror Devtron's JSON field names (camelCase, snake_case or PascalCase, depending on the
  endpoint);
- use `#[serde(default)]`;
- ignore unknown fields.

Domain types convert empty strings, `0` ids and missing fields into `Option::None` in one place,
so views never see a `""` that means "unknown".

## Local (tool-owned)

### InstanceConfig

| Field | Type | Rules |
|---|---|---|
| `name` | string | Unique within the registry. Pattern `^[a-z0-9][a-z0-9-]{0,31}$`. It is the registry key. |
| `url` | URL | `https` required (`http` only for `localhost`). The trailing `/` and any `/orchestrator` suffix are stripped when loading. |
| `auth` | `AuthConfig` | How the instance authenticates (constitution II). In this feature the only variant is `TokenFile { path }`, where `path` defaults to `~/.devtron-token` and a leading `~` is expanded. The file must exist and be readable. A permission warning fires when `mode & 0o077 != 0` (FR-005). Any other `kind` is refused on load. |
| `color` | optional named or hex colour | Used for the header, and later for confirmation dialogs (constitution I). |

The **Registry** holds `last_instance: Option<name>` and `instances: map<name, InstanceConfig>`.

- Saving writes a temp file, then renames it.
- Removing an instance never touches its credentials, for example its token file (FR-031).

### Credentials (runtime only, never persisted by the tool)

`Credentials` is what a `Session`'s client authenticates with. The rest of the code never sees
which auth kind produced it.

| Variant | Built from | Sent as |
|---|---|---|
| `ApiToken { secret: SecretString, claims: TokenClaims }` | `AuthConfig::TokenFile { path }`, read at connect time | header `token: <jwt>` |

Planned variants, not in this feature: `SessionCookie`, from a username/password login, and a
keyring-backed token. Each one only adds a variant and a builder.

`TokenClaims = { expires_at: Option<Timestamp> (JWT exp, decoded without verification), email: Option<String> (shown in the header as the user) }`.

`Credentials` has no `Display` implementation. Its `Debug` prints `Credentials(***)`. An error
message that needs the credentials' origin asks for `AuthConfig::describe()` (for example
"token file ~/.devtron-token"), never for the secret.

### Session

`{ instance: InstanceConfig, client: devtron_api::Client, cancel: CancellationToken, epoch: u64 }`

State transitions:

```text
         register/launch                 switch(other)
(none) ───────────────▶ Connecting ─────────────────────▶ cancel old; epoch += 1; Connecting
                          │   ▲
          check ok        │   │ retry
                          ▼   │
                        Connected ◀─────────── (views load)
                          │
          401/403/expired/unreachable
                          ▼
                     ConnectFailed{cause}  ── user opens :instances ──▶ Instance list
```

Every message from a background task carries its `epoch`. `update` drops a message whose epoch
does not equal `session.epoch` (FR-033, SC-011).

### Loadable<T> (every data view)

`Loading | Loaded(T) | Empty | Failed { cause, retryable: bool }`

- `Empty` is only used when the request succeeded with no rows (FR-020).
- A refresh (FR-022) keeps showing `Loaded(T)` while reloading. A spinner marks it as stale.

### LogBuffer (one per log viewer)

| Field | Rules |
|---|---|
| `lines` | Ring of at most 50,000 styled lines. On overflow the oldest is dropped and `dropped += 1` (FR-017). |
| `follow` | `true` initially. Set to `false` when the user scrolls up, and back to `true` when they reach the bottom or press `G` (FR-014). |
| `search` | Optional query, case-insensitive. Keeps a list of matching line indices, updated as lines arrive (FR-016). |
| `stages` | Markers from `STAGE_INFO` lines: `{stage, start, end?, status}` |
| `source_state` | `Streaming | Reconnecting{attempt} | Ended{final_status?} | Gone | Failed{cause}` |

## Remote (read from Devtron)

### Project

`{ id, name }`, from `GET /team`. Used to resolve `projectId` into a name.

### Application

| Field | Wire source (`POST /app/list` → `appContainers[]`) |
|---|---|
| `id` | `appId` |
| `name` | `appName` |
| `project` | `projectId` → Project name |
| `environments` | `environments[]` with a non-empty `environmentId` and `environmentName` → `[EnvRef {id, name, cluster, last_deployed_at?}]` |

The list endpoint returns Devtron apps only (`app_type = 0`, research R4).

### EnvironmentState (app detail row)

| Field | Wire source (`GET /app/other-env` → `result[]`, plus resource tree) |
|---|---|
| `env_id`, `env_name` | `environmentId`, `environmentName` |
| `cluster_id`, `cd_pipeline_id` | `clusterId`, `pipelineId` |
| `is_prod` | `prod` |
| `deployed` | `None` when `lastDeployed` is empty, meaning never deployed (FR-011). Otherwise `DeployedBuild`. |
| `health` | `Loadable<Health>` from `resource-tree.result.status`. Not requested when never deployed or when `deploymentAppDeleteRequest = true`, which shows as `DeletionPending` (R3). A 403 gives `Failed { cause: Forbidden }` for this row only (FR-035). |
| `last_deployment` | `Loadable<RunStatus>`: the outcome of the newest Deployment, built from A7 with `limit=3` (the newest runners, grouped by `cd_workflow_id`; the first group wins). Not requested when never deployed. Shown next to `health`, so a failed rollout is visible while the old pods are still healthy (FR-010, US2 scenario 3). |

`DeployedBuild = { image, tag (after last ':'), commit (commits[0], short), deployed_at (lastDeployed),
deployed_by (lastDeployedBy email), runner_id (latestCdWorkflowRunnerId) }`

`Health = Healthy | Progressing | Degraded | Suspended | Missing | Hibernating | Unknown(String)`,
parsed from the tree's `status`. Values not in the list are kept as `Unknown(text)`, never
rejected.

### Deployment (history entry)

Built from the runners (`GET /resource/history/deployment/cd-pipeline/v1` → `cdWorkflows[]`,
snake_case) grouped by `cd_workflow_id`:

| Field | Source |
|---|---|
| `id` | `cd_workflow_id` |
| `deploy` | Runner with `workflow_type = DEPLOY`. It can be missing if only PRE ran, or if the deployment was cut at a page boundary. |
| `stages` | Runners with `workflow_type ∈ {PRE, POST}`, ordered PRE → POST (FR-024) |
| `started_at` | Earliest `started_on` |
| `author` | `email_id` (falls back to `triggered_by` id) |
| `build` | `image` → tag. Commit from `ciMaterials` / `gitTriggers` when present. |
| `outcome` | DEPLOY runner `status`, else the last stage's status |

`RunStatus = Starting | Queued | Initiating | Running | Progressing | Succeeded | Failed | Aborted |
Cancelled | TimedOut | Unknown(String)`. Wire strings are matched ignoring case (`CANCELLED` and
`Cancelled` both appear).

**Merging across pages**: pages are counted in runners (limit 20). When a new page arrives, runners
whose `cd_workflow_id` already exists are merged into that Deployment instead of creating a
duplicate.

**Stage log is openable** when `pod_status` is set and not `Pending` (R6).

### CiPipeline

`{ id, name, parent_pipeline_id?, parent_app_id? }` from `GET /app/{appId}/ci-pipeline/min`.
Builds of a linked pipeline (`parentCiPipeline > 0`) are read from the parent pipeline (R7).

### Build

`GET /app/ci-pipeline/{pipelineId}/workflows` → `ciWorkflows[]` (camelCase):

| Field | Source |
|---|---|
| `id`, `pipeline_id` | `id`, `ciPipelineId` |
| `status` | `status` → `RunStatus` |
| `started_at`, `finished_at` | `startedOn`, `finishedOn` (an all-zero date means "not finished") |
| `author` | `triggeredByEmail` |
| `commit` | First `gitTriggers[*].Commit` (short), with `.Message` (first line) and `.Author`. Keys are PascalCase. |
| `artifact` | `artifact` (image) |
| `pod_status` | `podStatus`. Logs are empty while `Pending`. |

### Pod / Container

From `GET /app/detail/resource-tree` → `result.nodes[]` where `kind == "Pod"`, joined with
`result.podMetadata[]` by `name`:

| Field | Source |
|---|---|
| `name`, `namespace` | node `name`, `namespace` |
| `status` | info `Status Reason` |
| `ready` | info `Containers` (e.g. `1/1`) |
| `restarts` | info `Restart Count`. **Absent means 0** (kubelink only emits it when > 0, R9). |
| `created_at` → `age` | node `createdAt` |
| `containers` | `podMetadata.containers[]`. Init and ephemeral containers are left out of this feature. |

A pod log request also needs `cluster_id`, `deployment_type` (`helm` → 0, `argo_cd` → 1, from
`detail/v2.deploymentAppType`) and the app and environment ids.

## Relationships

```text
Registry 1─* InstanceConfig ── active ── Session 1─1 Client
Session ⟶ Project *, Application *
Application 1─* EnvironmentState 1─* Deployment 1─* Stage(runner) ─▶ Log
Application 1─* CiPipeline 1─* Build ─▶ Log
EnvironmentState 1─* Pod 1─* Container ─▶ Log (current | previous)
```
