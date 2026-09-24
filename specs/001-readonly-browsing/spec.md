# Feature Specification: Read-Only Browsing of a Devtron Instance

**Feature Branch**: `001-readonly-browsing`

**Created**: 2026-09-24

**Status**: Draft

**Input**: User description: "First version of a keyboard-driven terminal interface (in the style of
k9s / lazygit) for our Devtron instance, strictly read-only: list applications, open an application
to see its environments and what is deployed where, browse deployment history, and follow CI builds
with their live logs."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Connect and browse the application list (Priority: P1)

A DevOps engineer starts the tool from their terminal. On first use they register their Devtron
instance by giving it a short name (for example `prod`) and its address. The tool reads the engineer's
existing API token, confirms it works, and shows
every application the token can see, with enough information per row (name, project, environments it
is deployed to) to recognise the one they are looking for. They type a few characters to filter the
list down to a single application. A header shows the name of the instance they are connected to and
when its token expires.

**Why this priority**: Nothing else works without a verified connection and a way to find an
application. This alone is already useful: it answers "which apps exist, and where are they deployed?"
faster than opening the web interface.

**Independent Test**: With only this story built, launch the tool against the instance, filter the list
by part of a known application name, and check that the application appears with the correct project
and environments.

**Acceptance Scenarios**:

1. **Given** no instance is registered, **When** the user launches the tool, **Then** they are asked for
   a name and an address, the instance is saved, and later launches connect to it without asking.
2. **Given** a valid token and a reachable instance, **When** the tool starts, **Then** the application
   list is shown and the header shows the instance name and the token's expiry date.
3. **Given** the application list is shown, **When** the user starts a filter and types part of a name,
   **Then** only matching applications remain, and clearing the filter restores the full list.
4. **Given** the token has expired, **When** the tool starts, **Then** it says the token expired on
   a specific date and how to replace it, instead of a generic authentication error.
5. **Given** the token file can be read by other users on the machine, **When** the tool starts,
   **Then** it shows a warning naming the file and still continues.

---

### User Story 2 - See what is deployed where for one application (Priority: P2)

From the list, the engineer opens an application. They see one row per environment showing the
deployment status (for example healthy, progressing, degraded, failed), which build is deployed
(image tag and source commit), when it was deployed and by whom. They go back to the list with one
key and open another application.

**Why this priority**: "What version is running in production right now, and is it healthy?" is the
question engineers ask most often, and it is the main reason to reach for this tool.

**Independent Test**: Open a known application and compare, environment by environment, the status,
deployed build, time and author with what the Devtron web interface shows.

**Acceptance Scenarios**:

1. **Given** the application list, **When** the user opens an application, **Then** every environment it
   is configured for is listed with status, deployed build, deployment time and deploying user.
2. **Given** an environment where the application has never been deployed, **When** the detail view is
   shown, **Then** that environment is listed and clearly marked as never deployed.
3. **Given** the detail view is open, **When** the user presses the "back" key, **Then** they return to the
   application list with their previous filter and selection kept.
4. **Given** the detail view is open, **When** the user asks for a refresh, **Then** statuses are reloaded
   without leaving the view.

---

### User Story 3 - Browse deployment history (Priority: P3)

From an application's environment, the engineer opens its deployment history: a list of past
deployments, newest first, each with its time, author, deployed build and outcome. This lets them
answer "what changed, and when?" while investigating an incident.

**Why this priority**: It is essential when investigating incidents, but it is used less often than
checking the current state.

**Independent Test**: Open the history of an environment with known recent deployments and check that
their order, times, authors, builds and outcomes match the web interface.

**Acceptance Scenarios**:

1. **Given** an application's environment, **When** the user opens its history, **Then** past
   deployments are listed newest first with time, author, build and outcome.
2. **Given** a history longer than one screen, **When** the user scrolls past the loaded entries,
   **Then** older entries are loaded as needed without freezing the interface.
3. **Given** an environment with no deployments, **When** its history is opened, **Then** an explicit
   "no deployments yet" state is shown.
4. **Given** a deployment that ran pre- or post-deployment stages, **When** the user opens that
   entry, **Then** each stage is listed with its outcome and its log can be opened in the same log
   viewer as builds (User Story 4).
5. **Given** a deployment without pre- or post-deployment stages, **When** the user opens that entry,
   **Then** its outcome is shown and the view states that no stages ran.

---

### User Story 4 - Follow CI builds and their live logs (Priority: P4)

From an application, the engineer opens its build list: recent CI builds with their status (running,
succeeded, failed, cancelled), trigger time, author and source commit. They open a running build and
watch its log output arrive live, scroll back through it, and search it for a term. They can also open
a finished build to read its full log.

**Why this priority**: Following a build without keeping a browser tab open is a big gain in daily work,
but it builds on the navigation from the earlier stories and handles more data.

**Independent Test**: Trigger a build from the web interface, then open it in the tool and check that log
lines appear while the build runs and that the final status matches.

**Acceptance Scenarios**:

1. **Given** an application, **When** the user opens its build list, **Then** recent builds are listed
   newest first with status, trigger time, author and commit.
2. **Given** a running build, **When** the user opens its logs, **Then** new lines appear as the build
   produces them, and the view follows the end of the log unless the user has scrolled up.
3. **Given** a log view, **When** the user searches for a term, **Then** matches are highlighted and the
   user can jump between them.
4. **Given** a finished build, **When** the user opens its logs, **Then** the complete log is shown,
   along with the final status.
5. **Given** the live log connection drops while the build is still running, **When** this happens,
   **Then** the user is told and the tool tries to reconnect without losing the lines already shown.

---

### User Story 5 - Follow the runtime logs of an application's pods (Priority: P5)

From an application's environment, the engineer opens the list of running pods: name, status,
number of restarts and age. They open a pod and watch its logs live in the same log viewer as
builds. If the pod has several containers, they choose which one. If a container has restarted, they
can also read the logs from before the restart to see why it crashed.

**Why this priority**: During an incident, "what is the running application saying?" usually follows
straight after "what is deployed?" (User Story 2). It reuses the log viewer from User Story 4, so it
comes after that.

**Independent Test**: Open an environment of a known running application, compare its pod list with
the web interface, open a pod, and check that fresh log lines appear as the application produces
them.

**Acceptance Scenarios**:

1. **Given** an application's environment, **When** the user opens its pods, **Then** each running pod
   is listed with name, status, restart count and age.
2. **Given** a pod with a single container, **When** the user opens it, **Then** its logs stream live
   in the log viewer, following the end, with the same scroll and search behaviour as build logs.
3. **Given** a pod with several containers, **When** the user opens it, **Then** they choose a container
   before logs are shown, and can switch container without going back to the pod list.
4. **Given** a container that has restarted, **When** the user asks for the previous logs, **Then** the
   logs from before the last restart are shown.
5. **Given** the pod being followed is deleted or replaced, **When** this happens, **Then** the user is
   told the pod is gone, the lines already shown are kept, and they can go back to the pod list.
6. **Given** an environment with no running pods, **When** the pods view is opened, **Then** an
   explicit "no running pods" state is shown.

---

### User Story 6 - Register and switch between Devtron instances (Priority: P6)

The engineer works with more than one Devtron instance (for example `staging` and `prod`). From
inside the tool they register another instance with a name, an address, the token file to use for
it and an optional colour. They open the instance list, which shows each instance's name, address
and token expiry, and switch to another one. Every view then shows only data from the new instance,
and the header shows its name in its colour, so they always know which instance they are looking at.
They can also start the tool directly on a given instance.

**Why this priority**: It is only useful once there is more than one instance. Instances are named
from User Story 1 onward, so adding this later needs no change to existing setups.

**Independent Test**: Register two instances with different tokens, switch between them, and check
that the application list, header and colour change each time and that no data from the previous
instance remains visible.

**Acceptance Scenarios**:

1. **Given** one registered instance, **When** the user registers a second one with a name, address and
   token file, **Then** it appears in the instance list and is saved for later launches.
2. **Given** a name that is already registered, **When** the user tries to register it again, **Then**
   the tool refuses and says the name is taken.
3. **Given** several registered instances, **When** the user switches to another one, **Then** the tool
   connects to it, goes back to its application list, and shows its name and colour in the header.
4. **Given** a live log stream or a slow request on the current instance, **When** the user switches,
   **Then** that work stops, and nothing it would have returned appears in the new instance's views.
5. **Given** several registered instances, **When** the user starts the tool naming one of them,
   **Then** it connects to that instance. Without a name, it connects to the last instance used.
6. **Given** one instance's token has expired but not another's, **When** the user opens the instance
   list, **Then** the expired one is marked as such, and switching to it gives the expired-token
   message (User Story 1, scenario 4).
7. **Given** a registered instance, **When** the user removes it and confirms, **Then** it disappears
   from the list. Its token file is left untouched.

---

### Edge Cases

- The instance is unreachable or very slow: the interface stays usable, shows a loading state,
  then an error the user can retry. It never freezes or shows a blank screen.
- The token is valid but lacks permission for part of the data: views show what is allowed and
  state that the rest is not visible with this token.
- The instance has hundreds of applications or environments: listing and filtering stay smooth.
- An application has no environments configured: the detail view says so explicitly.
- A build log is very large (hundreds of thousands of lines): the tool keeps working, possibly
  dropping the oldest lines from memory, and says that it did.
- The terminal is resized, or is too small to display a view: the layout adapts, or a clear
  "terminal too small" message is shown.
- The instance returns data with fields missing or unexpected: affected cells are shown as
  unknown, and the rest of the view still loads.
- The tool is interrupted or crashes: the terminal is always returned to a usable state.
- The user switches instance while a request or live log is in progress on the current one: that
  work is abandoned, and its results never appear once the switch is done.
- The last-used instance was removed, or is unreachable at launch: the tool opens the instance list
  instead of failing, so the user can pick another.
- The user names an instance at launch that is not registered: the tool says so and lists the
  registered names.

## Requirements *(mandatory)*

### Functional Requirements

**Connection and credentials**

- **FR-001**: When no instance is registered, the tool MUST ask for a name and an address and save the
  instance in the user's configuration. From then on it MUST connect without asking.
- **FR-002**: The tool MUST read each instance's API token only from a token file set for that instance
  (by default the single token file in the user's home directory), or from an environment variable
  that applies to the active instance only. It MUST NOT accept a token as a command-line argument.
  Tokens MUST NOT be stored in the configuration.
- **FR-003**: The tool MUST check the token and connection at startup and on every instance switch,
  and report the cause of any failure separately: unreachable instance, invalid token, expired token
  (with the expiry date), insufficient permissions.
- **FR-004**: The tool MUST show the active instance's name, in its colour if one is set, and its
  token's expiry date at all times.

**Instances (US1, US6)**

- **FR-030**: Users MUST be able to register several instances, each with a unique name, an address, a
  token file and an optional colour. Registering MUST be possible from inside the tool.
- **FR-031**: Users MUST be able to remove a registered instance after confirming. Removing it MUST NOT
  touch its token file, and MUST NOT change anything on the Devtron instance.
- **FR-032**: The tool MUST list registered instances with name, address and token status (valid until
  a date, expired, or unreadable), and let the user switch to any of them.
- **FR-033**: Only one instance MUST be active at a time. Switching MUST stop all requests and log
  streams of the previous instance, and MUST discard its data. After a switch, no view may show data
  that came from another instance.
- **FR-034**: Users MUST be able to choose the instance at launch. Without a choice, the tool MUST use
  the last instance used. If that instance no longer exists or cannot be reached, it MUST open the
  instance list instead of exiting.
- **FR-005**: The tool MUST warn when the token file can be read by users other than its owner.
- **FR-006**: The token MUST NOT appear on screen, in error messages, or in any log or diagnostic output.

**Read-only guarantee**

- **FR-007**: This feature MUST NOT offer, or perform, any action that changes state on the Devtron
  instance. Every request it sends MUST be read-only.

**Application list (US1)**

- **FR-008**: The tool MUST list every application visible to the token, showing name, project and the
  environments each application is deployed to.
- **FR-009**: Users MUST be able to filter the list by typing part of a name. Matching MUST ignore case
  and update with each keystroke.

**Application detail (US2)**

- **FR-010**: For a selected application, the tool MUST list each configured environment with
  deployment status, deployed build (image tag and source commit), deployment time and deploying user.
- **FR-011**: Environments where the application was never deployed MUST be listed and marked as such.

**Deployment history (US3)**

- **FR-012**: For an application's environment, the tool MUST list past deployments newest first, with
  time, author, deployed build and outcome, and load older entries as the user scrolls.
- **FR-024**: For a deployment entry, the tool MUST list the pre- and post-deployment stages that ran,
  with their outcome, and let the user open each stage's log. If no stage ran, it MUST say so.

**Builds (US4)**

- **FR-013**: For a selected application, the tool MUST list recent CI builds newest first, with status,
  trigger time, author and source commit.
- **FR-015**: For a finished build, the tool MUST show its complete log and final status.

**Pods (US5)**

- **FR-025**: For an application's environment, the tool MUST list the application's pods with name,
  status, restart count and age.
- **FR-026**: For a pod, the tool MUST stream the logs of a container. When a pod has several
  containers, the user MUST choose the container and be able to switch without leaving the log view.
- **FR-027**: For a container that has restarted, users MUST be able to view the logs from before its
  last restart.
- **FR-028**: When a followed pod disappears, the tool MUST say so and keep the lines already shown.

**Log viewer (US3, US4, US5)**

- **FR-014**: For anything still running (a build, a deployment stage, a pod), the log viewer MUST show
  output as it is produced. It MUST follow the end of the log and stop following when the user scrolls
  up, until they return to the bottom.
- **FR-016**: Users MUST be able to search within a log view and jump between matches.
- **FR-017**: The tool MUST limit how much log output it keeps in memory, and say when older lines
  have been dropped.
- **FR-029**: Build logs, deployment stage logs and pod logs MUST all open in the same log viewer, with
  the same keys and behaviour.

**Navigation and responsiveness (all stories)**

- **FR-018**: All navigation MUST be possible with the keyboard alone and follow one grammar across all
  views: switch view by command, filter, open, go back, help, quit.
- **FR-019**: Each view MUST show the keys available in it, and a help screen MUST list all keys.
- **FR-020**: Each view MUST have distinct loading, empty and error states. An error MUST be
  retryable without restarting the tool.
- **FR-021**: The interface MUST keep responding to the keyboard while data is loading or the instance
  is unreachable.
- **FR-022**: Users MUST be able to refresh the current view on demand.
- **FR-023**: When the tool exits for any reason, including a crash, the terminal MUST be left in a
  usable state.

### Key Entities

- **Instance**: a Devtron deployment registered in the tool. It has a unique name, an address, a
  token file, an optional colour and, for the current user, a token with an expiry date. Exactly one
  instance is active at a time.
- **Project**: a grouping of applications within Devtron.
- **Application**: a deployable unit. It belongs to one project and is configured for one or more
  environments.
- **Environment**: a deployment target (for example staging or production), tied to a cluster.
- **Deployment**: one rollout of a build of an application to an environment. It has a time, author,
  build and outcome. The latest deployment defines the environment's current state.
- **Build**: one CI run of an application. It has a status, trigger time, author, source commit and
  the image it produced.
- **Deployment stage**: a pre- or post-deployment step that ran as part of one deployment, with an
  outcome and a log.
- **Pod**: a running instance of an application in an environment. It has a status, a restart count
  and an age, and contains one or more containers.
- **Container**: a process inside a pod that produces logs. It may have logs from before its last
  restart.
- **Log**: the ordered output of a build, a deployment stage or a container, which keeps growing
  while its source runs.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: On an instance with 200 applications, a user already set up reaches a filtered
  application list within 3 seconds of launching the tool.
- **SC-002**: A user can find out which build is running in a given environment of a given application
  in under 10 seconds from launch, with no mouse and no browser.
- **SC-003**: While a build, deployment stage or pod is running, a new log line appears in the tool
  within 2 seconds of appearing in the Devtron web interface.
- **SC-009**: During an incident, a user can go from launch to the live logs of a production pod of a
  given application in under 15 seconds, using only the keyboard.
- **SC-004**: When the instance is slow or unreachable, a keypress (navigation, back, quit) still takes
  visible effect immediately. The interface never appears frozen.
- **SC-005**: Across the full test suite and a week of daily use, zero requests that change state are
  sent to any Devtron instance.
- **SC-010**: A user with two registered instances can switch from one to the other and see the new
  instance's application list in under 5 seconds.
- **SC-011**: In 100% of switch tests, including switches during a live log stream or a slow request,
  no data from the previous instance appears after the switch.
- **SC-006**: The token is never visible in any screen, message, log or crash output produced by the
  tool, checked by searching all captured output for the token value.
- **SC-007**: For 100% of exits (normal, error, interrupt, crash), the terminal is left usable without
  running `reset`.
- **SC-008**: Every connection failure (unreachable, invalid token, expired token, missing permission)
  produces a message that names the cause, so the user knows what to fix without further investigation.

## Assumptions

- The tool is connected to one Devtron instance at a time, and the user switches between registered
  instances. Views that combine data from several instances at once are out of scope and not planned.
- Editing a registered instance is done by removing it and registering it again, or by editing the
  configuration file directly. A dedicated edit screen is not needed.
- Users already have a Devtron API token (generated from Devtron's global configuration). Generating or
  renewing tokens from the tool is out of scope.
- The tool sees exactly what the token is allowed to see. It adds no access control of its own.
- The target environment is a Linux terminal (including WSL2) with at least 80×24 characters and
  colour support.
- Headless commands with machine-readable output are out of scope for this feature. They will be
  specified separately.
- Any action that changes state (triggering builds, deploying, rolling back, restarting pods, editing
  configuration) is out of scope. It will be specified as a later feature under the constitution's
  confirmation rules.
- "Recent builds" means the latest builds the instance returns by default, with older ones loaded
  on scroll.
- Only Devtron-managed applications (with CI/CD pipelines) are in scope. Helm apps from the chart
  store and Devtron jobs are not listed by this feature. They are recorded as future features in the
  repository's `backlog.md`.
- Pod logs come through the Devtron instance, with the token's permissions. The user does not need
  direct access to the clusters.
- "Pods of an application" means the pods Devtron associates with that application in that
  environment. Other workloads in the same namespace are not shown.

## Clarifications

### Session 2026-09-24

- Q: Which application kinds are in scope? → A: Devtron-managed apps only. Helm apps and jobs are
  deferred to future features (see `backlog.md`).
- Q: Are pod runtime logs part of this feature, or only CI build logs? → A: Both. Added User Story 5
  and FR-025 to FR-028.
- Q: Can deployment pre/post stage logs be opened from the history? → A: Yes. Added acceptance
  scenarios 4–5 to User Story 3 and FR-024. They reuse the shared log viewer (FR-029).
- Q: Can the tool work with several Devtron instances? → A: Yes, by switching (k9s-context style), one
  active at a time. Aggregated views across instances are not planned. Added User Story 6, FR-030 to
  FR-034, SC-010 and SC-011, and amended FR-001, FR-002 and FR-004.
