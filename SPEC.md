# Agent Orchestrator Specification

**Date:** 2026-03-12
**Status:** Draft
**Owners:** Celest / Rudex / Slack integration work

## Summary

Celest already has three useful building blocks:

- A Slack-facing router with slash commands, interactive message handling, Rudex access, and a small admin API
- A scheduler service with a simple internal REST API and SQLite-backed state
- OpenCode and plugin skills as the user-facing control surface inside Celest

This spec defines a new `agent-orchestrator` service that becomes the stable source of truth for team execution on Rudex VMs. Slack, Celest skills, and future automations will all talk to this service instead of directly owning orchestration state.

The orchestrator will:

- Allocate and track Rudex VMs
- Launch and monitor OMC team runs on those VMs
- Normalize runtime state into a stable API
- Publish progress and status to Slack through the existing Slack router
- Expose a clean API for a Celest skill and future internal tooling

The Slack router remains the Slack edge. The orchestrator owns runtime state.

## Motivation

We want a workflow where users can:

- Start a team run from Slack with slash commands or modals
- See which teams are running on which Rudex VMs
- Inspect current status, failures, and recent progress
- Stop, retry, or sync a run without SSH-ing into infrastructure
- Use the same control surface from a Celest skill, not just from Slack

We also want a design that can absorb inspiration from:

- `oh-my-claudecode` as the execution engine for team orchestration
- `openai/symphony` as the system-level model for isolated runs, evidence, and reconciliation
- Slack App Home, threads, and Block Kit as the operator UI

## Goals

- Create a stable internal REST API for agent-team orchestration
- Decouple Slack-specific handling from orchestration state and business logic
- Support OMC team execution on Rudex VMs
- Provide a first-class status model for teams, workers, VMs, and artifacts
- Support both Slack control and Celest skill control through the same API
- Make status visible in Slack App Home, slash-command responses, and run threads
- Keep the first version simple enough to ship with SQLite and polling

## Non-Goals

- Replacing the current Slack router
- Replacing OMC with a native Celest team runtime
- Building a general-purpose workflow engine in v1
- Real-time streaming across every terminal pane in v1
- Full RBAC or multi-tenant isolation beyond current Celest controls
- Deep Rudex feature coverage beyond what is needed for VM placement and lifecycle

## Existing System Anchors

The orchestrator should fit the current codebase, not bypass it.

- `docker/docker-compose.yaml` already defines `slack-bot-router` and `scheduler`
- `opencode/packages/slack-bot-router/src/index.ts` already handles:
  - Rudex-backed commands such as `/vmstatus` and `/rudex`
  - Slack interactivity
  - an admin API for posting messages and channel management
- `apps/scheduler/src/api.ts` is a good model for a small internal REST API
- `plugins/celest-devops/skills/manage-tasks/SKILL.md` is a good model for a Celest skill backed by a service API

## Proposed Architecture

### High-Level Shape

```text
Slack UI
  -> slack-bot-router
     -> agent-orchestrator
        -> Rudex
        -> VM remote helper / SSH adapter
        -> OMC team runtime
        -> SQLite state + reconciliation

Celest skill
  -> agent-orchestrator
```

### Component Responsibilities

#### Slack Router

The Slack router stays responsible for:

- Socket Mode connectivity
- slash commands
- modals and interactive callbacks
- App Home publishing
- thread replies and message updates
- Slack auth and manifest sync

The router must not become the source of truth for team-run state.

#### Agent Orchestrator

The new service is responsible for:

- creating and tracking team runs
- VM assignment and lease tracking
- execution state normalization
- polling and reconciliation
- artifact indexing
- event history
- internal API auth and request validation

#### Rudex Adapter

The Rudex adapter is responsible for:

- listing candidate VMs
- creating or leasing VMs
- reading VM metadata and status
- releasing or terminating VMs when runs finish

#### OMC Adapter

The OMC adapter is responsible for:

- starting team runs on a VM
- reading team status
- stopping team runs
- collecting output and artifacts

The v1 adapter can use SSH plus shell commands and tmux/psmux-friendly helpers. A dedicated remote helper can be added later if needed.

### Local-First Runtime Slice

Before full OMC team orchestration is stable, the orchestrator should support a narrower but real execution mode:

- one single-pane `claude --dangerously-skip-permissions` instance per run
- hosted inside a Rudex-provisioned Windows VM
- controlled over SSH with `tmux` / `psmux`
- normalized into the same `TeamRun` lifecycle as future OMC-backed runs

This is the first real runtime target because it proves:

- Rudex VM allocation
- Windows VM preflight
- SSH command execution from the orchestrator container
- durable session control across separate SSH calls
- prompt submission, capture, and stop semantics without building the full team runtime yet

## Core Design Principles

### 1. Orchestrator Owns State

Slack messages are views, not storage. Router memory is cache, not truth.

### 2. Reconciliation Over Assumption

Runs can drift from desired state because of restarts, failed SSH sessions, VM preemption, or manual intervention. The orchestrator must periodically compare:

- database state
- Rudex state
- OMC runtime state

and repair mismatches.

### 3. Normalize Runtime State

Slack and Celest should not parse raw OMC or terminal text to decide what is happening. The orchestrator provides normalized states and summaries.

### 4. API-First

Everything the Slack bot can do should also be reachable through the orchestrator API so Celest skills and automations can use the same surface.

## Execution Model

### Team Run Lifecycle

Each user-triggered orchestration creates a `TeamRun`.

Lifecycle:

1. Request accepted
2. Team spec validated
3. VM selected or provisioned
4. Runtime bootstrapped on VM
5. OMC team launched
6. Polling and event capture begin
7. Slack thread and App Home views update as status changes
8. Run reaches terminal state or is stopped
9. VM is retained, released, or destroyed according to policy

### OMC Mapping

OMC concepts should be mapped into orchestrator concepts.

- OMC team invocation becomes a `TeamRun`
- OMC workers become `Workers`
- OMC phases such as plan, prd, exec, verify, and fix become normalized `RunPhase` values
- OMC outputs and session artifacts become `Artifacts`

The orchestrator should not expose OMC internals directly as its public model.

### Windows Single-Pane Claude Contract

The first non-mock runtime contract is:

1. Allocate or select a Rudex VM
2. Preflight the VM:
   - `where tmux`
   - `tmux -V`
   - `where claude`
   - verify workspace root such as `C:\ws`
3. Reject known-bad `psmux` builds rather than pretending launch succeeded
4. Bootstrap a single-pane tmux session via an attached SSH TTY:
   - `tmux new-session -s <session> -- cmd /K "cd /d C:\ws && claude --dangerously-skip-permissions"`
5. Treat timeout during attached bootstrap as acceptable if later `tmux list-panes -t <session>` proves the session exists
6. Never depend on `%pane_id` targets on Windows; target the bare session name instead
7. Capture the initial pane and:
   - auto-accept the bypass-permissions warning if present
   - otherwise proceed once an interactive Claude prompt is visible
8. Submit the user prompt with `tmux send-keys -t <session> -l -- <prompt>` and `C-m`
9. Read output with `tmux capture-pane -t <session> -p -S -<lines>`
10. Stop with `Ctrl-C` followed by `tmux kill-session`

This contract is intentionally single-pane. Multi-pane worker orchestration can be layered on later, after the Windows `psmux` control path is stable.

## Data Model

The following entities are required in v1.

### TeamSpec

Immutable description of requested work.

Fields:

- `id`
- `requestedByUserId`
- `requestedByUserName`
- `source` (`slack`, `celest-skill`, `api`, `scheduler`)
- `workspaceSlug`
- `projectPath`
- `agentFamily` or `runtime` (`omc`)
- `providerMix` (`claude`, `codex`, `gemini`, mixed)
- `workerPlan`
- `prompt`
- `settingsJson`
- `createdAt`

### TeamRun

Mutable runtime record.

Fields:

- `id`
- `teamSpecId`
- `status`
- `phase`
- `summary`
- `vmLeaseId`
- `runtimeHandle`
- `slackChannelId`
- `slackThreadTs`
- `appHomeUserIdsJson`
- `startedAt`
- `finishedAt`
- `lastObservedAt`
- `lastHealthyAt`
- `failureReason`
- `createdAt`
- `updatedAt`

Suggested statuses:

- `queued`
- `allocating_vm`
- `bootstrapping`
- `starting`
- `running`
- `verifying`
- `completed`
- `failed`
- `stopping`
- `stopped`
- `lost`

Suggested phases:

- `plan`
- `prd`
- `exec`
- `verify`
- `fix`
- `cleanup`
- `unknown`

### Worker

Normalized worker/process information.

Fields:

- `id`
- `teamRunId`
- `provider`
- `role`
- `index`
- `status`
- `vmName`
- `sessionName`
- `paneName`
- `startedAt`
- `finishedAt`
- `lastHeartbeatAt`
- `summary`

### VmLease

Represents the VM assigned to a run.

Fields:

- `id`
- `teamRunId`
- `rudexTaskId`
- `vmName`
- `vmIp`
- `region`
- `leaseMode` (`shared`, `exclusive`)
- `status`
- `createdAt`
- `releasedAt`
- `metadataJson`

### RunEvent

Append-only audit trail.

Fields:

- `id`
- `teamRunId`
- `kind`
- `severity`
- `message`
- `detailsJson`
- `source`
- `observedAt`

Examples:

- VM allocated
- OMC launch command started
- team phase changed to verify
- SSH health check failed
- artifact discovered
- manual stop requested

### Artifact

Pointer to evidence or useful outputs.

Fields:

- `id`
- `teamRunId`
- `kind`
- `label`
- `path`
- `url`
- `mimeType`
- `sizeBytes`
- `metadataJson`
- `createdAt`

Examples:

- OMC session summary
- replay log
- patch or diff
- CI result link
- PR link
- screenshot
- rendered terminal image

### SlackBinding

Slack presentation state for a run or dashboard target.

Fields:

- `id`
- `teamRunId`
- `surface` (`thread`, `app_home`, `channel_message`, `ephemeral`)
- `channelId`
- `threadTs`
- `messageTs`
- `userId`
- `lastPublishedHash`
- `updatedAt`

## Storage

### v1 Storage Choice

Use SQLite, following the scheduler pattern.

Benefits:

- low operational overhead
- easy local development
- enough for current scale
- already familiar in this repo

### Persistence Rules

- `RunEvent` is append-only
- `TeamSpec` is immutable after creation
- `TeamRun`, `Worker`, and `VmLease` are mutable and updated by reconciliation
- stale state must be repairable from Rudex and OMC observations

## REST API

The orchestrator exposes an internal API on the Docker network. Slack router and Celest skills are first-class clients.

All endpoints return JSON.

### Health

#### `GET /health`

Returns:

- service health
- DB connectivity
- Rudex adapter health
- OMC adapter health
- counts of active runs, VMs, and recent failures

### Team Runs

#### `POST /teams`

Creates a new `TeamSpec` and `TeamRun`.

Request body:

- `workspaceSlug`
- `projectPath`
- `prompt`
- `providerMix`
- `workerPlan`
- `requestedBy`
- `slackContext` optional
- `launchPolicy` optional

Response:

- created `TeamRun`
- selected launch policy
- initial status

#### `GET /teams`

List runs with filters:

- `status`
- `requestedBy`
- `workspaceSlug`
- `vmName`
- `limit`

#### `GET /teams/:id`

Returns:

- team run
- worker list
- VM lease
- latest events
- artifacts

#### `POST /teams/:id/stop`

Requests graceful stop.

#### `POST /teams/:id/kill`

Requests force stop. Admin-only in v1.

#### `POST /teams/:id/sync`

Forces immediate reconciliation for one run.

#### `POST /teams/:id/retry`

Creates a new run from the same spec, optionally with overrides.

### VMs

#### `GET /vms`

Returns normalized VM information:

- lease state
- owner run
- Rudex state
- health
- recent activity

#### `GET /vms/:id`

Detailed VM view.

#### `POST /vms/:id/release`

Explicit release when policy allows.

### Events and Artifacts

#### `GET /events`

Filterable event stream.

#### `GET /teams/:id/events`

Run-scoped events.

#### `GET /teams/:id/artifacts`

Run-scoped artifacts.

### Slack Helpers

These endpoints exist for clean router integration.

#### `GET /slack/home/:userId`

Returns a view model for Slack App Home.

#### `GET /slack/thread/:teamRunId`

Returns a view model for the run thread.

#### `POST /slack/actions`

Optional adapter endpoint if we want Slack action routing centralized in the orchestrator instead of the router. Not required in v1.

## API Authentication

The orchestrator is an internal service. v1 should still require a token for non-health routes.

Environment:

- `AGENT_ORCHESTRATOR_API_TOKEN`

Clients:

- Slack router uses bearer auth
- Celest skill uses bearer auth
- future scheduler integration uses bearer auth

## Slack Product Surface

### v1 Surfaces

Use the following Slack surfaces in v1:

- slash commands for mutations and quick actions
- App Home for dashboard status
- run threads for detailed progress
- ephemeral responses for command acknowledgements and errors

### Commands

Recommended commands:

- `/team-create`
- `/team-list`
- `/team-status`
- `/team-stop`
- `/team-retry`
- `/team-sync`
- `/vm-list`
- `/vm-status`

If command count is a concern, these can be grouped under:

- `/team ...`
- `/vm ...`

### App Home

App Home should be the main operator dashboard.

Sections:

- Active runs assigned to the current user
- Active runs for the team or workspace
- VM inventory summary
- Recent failures
- Quick actions

Group active runs by VM so users can answer:

- what is running
- where it is running
- whether it is healthy
- what phase it is in

### Run Thread

Each team run should have a canonical Slack thread.

Thread contents:

- current status summary
- VM info
- worker/provider summary
- recent events
- artifact links
- stop/retry/sync buttons

### Block Kit Strategy

Use snapshot-style Block Kit updates first.

Do not block the project on full streaming in v1.

Later enhancements can use:

- Slack AI app assistant statuses
- task cards
- plan blocks
- streaming text APIs

## Celest Skill Integration

Create a new skill, similar in spirit to `manage-tasks`, that speaks only to the orchestrator API.

Suggested skill name:

- `manage-teams`

Responsibilities:

- create runs
- list runs
- inspect a run
- stop or retry a run
- summarize failures
- answer questions using orchestrator data and artifacts

The skill must not shell directly to Rudex or OMC when the orchestrator can answer the question.

## Orchestrator Internal Modules

Suggested package layout:

```text
opencode/packages/agent-orchestrator/
  src/
    index.ts
    api.ts
    storage.ts
    types.ts
    reconcile.ts
    scheduler.ts
    slack-view-models.ts
    adapters/
      rudex.ts
      omc.ts
      ssh.ts
    services/
      team-runs.ts
      vm-leases.ts
      events.ts
      artifacts.ts
```

### Module Responsibilities

- `api.ts`: HTTP routes and auth
- `storage.ts`: SQLite schema and queries
- `reconcile.ts`: drift detection and repair
- `scheduler.ts`: periodic polling loop
- `slack-view-models.ts`: App Home and thread render payloads
- `adapters/rudex.ts`: Rudex commands and normalization
- `adapters/omc.ts`: OMC launch/status/stop contract
- `adapters/ssh.ts`: low-level remote execution

## Runtime and Polling Model

### Polling

Use polling in v1.

Poll sources:

- Rudex VM status
- remote process or tmux session status
- OMC status command output
- artifact directories or known files

Suggested default intervals:

- active runs: every 5 to 10 seconds
- completed or failed runs awaiting cleanup: every 30 to 60 seconds
- global VM inventory refresh: every 15 to 30 seconds

### Reconciliation Rules

Examples:

- DB says `running`, Rudex says VM gone -> mark run `lost` and emit event
- DB says `allocating_vm`, Rudex shows VM ready -> advance to `bootstrapping`
- DB says `running`, OMC process gone, exit code successful -> mark `completed`
- DB says `running`, OMC missing and no success evidence -> mark `failed` or `lost` based on diagnostics

## VM Policies

The orchestrator must support a simple policy model.

### Launch Policies

- `allocate_new`
- `reuse_idle`
- `prefer_workspace_pool`

### Lease Policies

- `release_on_complete`
- `retain_for_debug`
- `retain_until_timeout`

### Isolation Modes

- `exclusive` for sensitive or long-running work
- `shared` for trusted internal tasks on a shared VM pool

## Failure Handling

The orchestrator must produce explicit failure reasons whenever possible.

Examples:

- Rudex allocation failed
- VM boot timeout
- SSH connection failed
- OMC command launch failed
- OMC status unreadable
- runtime became unreachable
- artifact collection failed

Each failure should:

- update `TeamRun.failureReason`
- append a `RunEvent`
- trigger a Slack update if the run has Slack bindings

## Slack Publication Rules

Slack updates should be rate-limited and state-based.

- Publish immediately on creation
- Publish immediately on terminal state
- Publish on important phase changes
- Coalesce noisy polling updates

Use `lastPublishedHash` in `SlackBinding` to avoid redundant updates.

## Security

### Secrets

Use environment variables, following current service patterns.

Likely values:

- orchestrator API token
- optional Rudex API base URL
- optional Slack router admin token if the orchestrator ever posts through the router

Provisioning, archive, Windows VM SSH, and Claude bootstrap should run through the user workspace's Rudex and SSH context rather than shared orchestrator credentials.

### Access Boundaries

- Slack router and Celest skill are trusted internal clients
- direct public exposure is out of scope
- force-kill and VM-destroy actions should be protected more tightly than read-only actions

### Auditability

All mutating actions should emit `RunEvent` entries with:

- actor
- source surface
- requested action
- resulting state

## Deployment

### Docker

Add a new service alongside `scheduler`.

Suggested container:

- `agent-orchestrator`

Suggested port:

- `3004`

Suggested data path:

- `/data/agent-orchestrator`

### Build Pattern

Follow the scheduler model:

- service lives in `opencode/packages/agent-orchestrator`
- dedicated Dockerfile at `docker/Dockerfile.agent-orchestrator`
- Compose wiring in `docker/docker-compose.yaml`

## Initial Slack Router Changes

The router should be extended, not rewritten.

v1 changes:

- add orchestrator API client
- add team and VM slash commands or grouped command routing
- publish App Home using orchestrator view models
- post thread updates using orchestrator data

The existing Rudex commands can remain during migration, but new team lifecycle actions should route through the orchestrator.

## Observability

The orchestrator should log structured events for:

- incoming API requests
- state transitions
- Rudex calls
- OMC calls
- Slack publication attempts
- reconciliation results

Useful metrics:

- active runs
- runs by status
- mean allocation time
- mean completion time
- failed reconciliations
- lost runs
- VM utilization

## Phased Delivery Plan

### Phase 1: Core Service

- create package scaffold
- SQLite schema
- health endpoint
- create/list/get/stop/sync team-run endpoints
- Rudex adapter for VM allocation and status
- OMC adapter for start/status/stop
- polling reconciler

### Phase 2: Slack Integration

- router client for orchestrator
- slash commands for team lifecycle
- run-thread status updates
- App Home dashboard

### Phase 3: Celest Skill

- `manage-teams` skill
- operator-facing workflows
- run inspection and failure summarization

### Phase 4: Hardening

- retry policies
- richer artifact collection
- lease retention policies
- improved failure classification
- admin-only destructive actions

### Phase 5: Advanced UX

- Slack AI app features
- streaming task cards or plan blocks
- scheduler-triggered team runs
- policy-driven placement and priority queues

## Open Questions

- What exact OMC CLI contract should the orchestrator rely on for `status` and `shutdown`?
- Should the orchestrator SSH into the VM directly in v1, or should we add a tiny remote helper early?
- Do we want one shared VM pool or per-workspace pools first?
- Should completed runs retain their VM by default for debugging, or default to release?
- How much of the existing `/vmstatus` and `/rudex` experience should remain as raw escape hatches?
- Do we want App Home as the primary dashboard immediately, or start with thread snapshots only?

## Recommendation

Build the orchestrator as a small, boring service first:

- SQLite
- polling
- REST API
- Rudex + SSH + OMC adapters
- Slack snapshot updates

Do not start with a grand event bus or streaming-first architecture. The key decision is organizational, not technical:

`agent-orchestrator` must become the single source of truth for team execution state.

Once that boundary exists, Slack UX, Celest skills, scheduler triggers, and richer evidence collection can all be layered on top without reworking the control plane.
