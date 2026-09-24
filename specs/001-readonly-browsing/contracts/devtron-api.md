# Contract: Devtron HTTP endpoints used (the read-only allowlist)

This table **is** the allowlist in `crates/devtron-api/src/allowlist.rs`.

- A request whose method and path don't match a row is refused before it is sent (research R3).
- Adding a row requires a scrubbed fixture and a model test (constitution V). A row with a method
  other than `GET` also requires a note proving that it is a pure query.

Paths are relative to `{instance_url}/orchestrator`. Every request sends `token: <jwt>`. JSON
responses use the envelope `{code, status?, result?, errors?}`, and errors are classified by HTTP
status (R2).

| # | Method | Path pattern | Query / body | Used by | Fixture |
|---|---|---|---|---|---|
| A1 | GET | `/user/check/roles` | none | connection check, FR-003 | `check-roles.json` |
| A2 | POST | `/app/list` | body `{offset, size, sortBy:"appNameSort", sortOrder:"ASC"}`. **Pure query** (`AppListingRestHandler.go:304-437`). | apps view, FR-008 | `app-list.json` |
| A3 | GET | `/team` | none | project names | `team.json` |
| A4 | GET | `/app/other-env` | `app-id` | app detail, FR-010/011 | `other-env.json` |
| A5 | GET | `/app/detail/v2` | `app-id`, `env-id` | namespace and `deploymentAppType` for pods | `detail-v2.json` |
| A6 | GET | `/app/detail/resource-tree` | `app-id`, `env-id`. **Never sent when `deploymentAppDeleteRequest` is true** (R3). | health, pods, FR-025 | `resource-tree.json` |
| A7 | GET | `/resource/history/deployment/cd-pipeline/v1` | `filterCriteria=application/devtron-application\|id\|{appId}`, `filterCriteria=environment\|id\|{envId}`, `offset`, `limit` | history (FR-012), and latest deployment outcome per environment with `limit=3` (FR-010) | `history.json` |
| A8 | GET | `/app/cd-pipeline/workflow/logs/{appId}/{envId}/{pipelineId}/{wfrId}` | `followLogs` (bool). SSE. | stage logs, FR-024 | `cd-stage-logs.sse` (synthesized, R6) |
| A9 | GET | `/app/{appId}/ci-pipeline/min` | none | builds view pipeline selector | `ci-min.json` |
| A10 | GET | `/app/ci-pipeline/{pipelineId}/workflows` | `offset`, `size` (both required) | builds, FR-013 | `ci-workflows.json` |
| A11 | GET | `/app/ci-pipeline/{pipelineId}/workflow/{workflowId}/logs` | `followLogs` (bool). SSE. | build logs, FR-014/015 | `ci-logs.sse` |
| A12 | GET | `/k8s/pods/logs/{podName}` | `containerName`, `follow=true`, `previous`, `appId={clusterId}\|{appId}\|{envId}`, `appType=0`, `deploymentType`, `namespace`, `tailLines=500`. SSE. | pod logs, FR-026/027 | `pod-logs.sse` |

## Stream contracts

**A8 and A11** (CI and CD-stage logs):

```text
id:0 / data:START_OF_STREAM          event:START_OF_STREAM
id:<n> / data:<line>                  (one frame per log line; may contain ANSI; may start STAGE_INFO|{json})
data:END_OF_STREAM                    event:END_OF_STREAM         → LogEvent::End
UNEXPECTED_END_OF_STREAM                                          → reconnect
event:RECONNECT_STREAM                (reply to a Last-Event-ID resume)
```

A JSON envelope (400/500) instead of `text/event-stream` means the logs are unavailable. For
example, `userMessage: "logs-not-stored-in-repository"` shows as "logs are not archived on this
instance".

**A12** (pod logs):

```text
id: <unixNanos> / data:<line>         (note: space after "id:")
event:PING                            every 30 s, ignored
event:CUSTOM_ERR_STREAM / data:<err>  → LogEvent::Error(err)
(connection close, no END frame)      → re-check A6: pod missing → Gone, else reconnect with Last-Event-ID
```

A non-200 status is always fatal for the stream, even if bytes follow (R8).

## Error classification (shared by all rows)

| Observation | `ApiError` | Shown to user as |
|---|---|---|
| DNS, TLS or connection failure, or timeout | `Unreachable` | "cannot reach {instance}: {reason}" (retryable) |
| 401 and the credentials' local expiry (the token's `exp`) is in the past | `Expired { at }` | "credentials for {instance} expired on {date}", plus a fix hint from the auth kind (token file: "replace {path}") |
| 401 otherwise | `Unauthorized` | "credentials for {instance} were rejected" |
| 403 | `Forbidden` | "not visible with your access". A 403 on one row or sub-request marks only that part, and the rest of the view renders (FR-035). |
| 404 | `NotFound` | "not found", for example an app deleted since the list loaded (retryable) |
| other ≥ 400 | `Server { status, message }` | the message from R2 (retryable) |
| JSON that fails to parse | `Decode { endpoint }` | "unexpected response from {endpoint}". Only a malformed envelope can cause this, since fields are tolerant. |

Messages never include credentials, request headers, or the full request URL with its query string.
