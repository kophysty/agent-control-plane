# ADR-0001 — Platform strategy: GitHub + Nimbalyst + Agentplane

- **Status:** Accepted for validation spikes
- **Date:** 2026-08-24
- **Decision owner:** project owner
- **Implementation repository:** `kophysty/agent-control-plane`

## 1. Context

Current Agent Team attempted to combine work planning, inter-agent communication, liveness, process supervision, provider invocation, audit logging and operator visibility through Markdown inboxes, PowerShell runners and Git.

It accumulated valuable contracts, but failed the primary user requirement: **all background work must remain visible, attributable and automatically preserved**.

A new full desktop platform would solve the ownership problem but duplicate a large amount of functionality already present in Nimbalyst and Agentplane.

## 2. Decision

Build **a thin local Agent Control Plane**, not a new general-purpose agent application.

### Selected responsibilities

```text
GitHub Issues / Projects
  owns business work queue, priority, routing fields and business status

Agent Control Plane
  owns local claim, lease, run mapping, reconciliation and external-effect outbox

Agentplane
  owns one issue's deterministic workflow/evidence contract

Nimbalyst
  owns provider sessions, raw transcript, tool stream, workstreams, worktrees and operator UI

Git
  owns source changes, branch, commits and diff
```

The new repository contains adapters, policies, a small daemon/CLI/MCP and later a Nimbalyst panel. It does not contain a replacement chat client or model runtime.

## 3. Why this minimizes development

### What is already available in Nimbalyst

- Claude Code and Codex provider integration;
- persisted provider-native transcript;
- nested subagent presentation;
- sessions, archive, search and resume;
- parallel sessions and workstreams;
- worktree management;
- child-session orchestration via MCP;
- queued prompts and interactive responses;
- notification to desktop/mobile;
- extension system and panels.

### What is already available in Agentplane

- typed planner/executor/evaluator episodes;
- state fingerprints and stale-result rejection;
- worktree/branch/PR route;
- approvals and human boundaries;
- verification receipts independent of agent claims;
- rework loop;
- findings, rollback and ACR;
- recovery/effect-in-doubt semantics.

### What Symphony specifies well

- normalized issue model;
- `dispatchable` eligibility;
- polling/reconciliation;
- concurrency and priority;
- per-issue workspace;
- retry/backoff;
- stop when issue becomes ineligible;
- repository-owned workflow configuration.

Our implementation is limited to the missing seams.

## 4. Component model

```text
┌──────────────────────────────────────────────────────────────┐
│ GitHub Project                                               │
│ Issues · Priority · Phase · Team · Status · Dependencies     │
└──────────────────────────┬───────────────────────────────────┘
                           │ GitHub App/API
                           ▼
┌──────────────────────────────────────────────────────────────┐
│ acpd — local daemon                                          │
│                                                              │
│ GitHub adapter       Scheduler        Reconciler             │
│ Lease manager        Outbox           Archive coordinator    │
│ Agentplane bridge    Nimbalyst links  Read API / MCP          │
│                                                              │
│ SQLite: structured runtime only                              │
└───────────────┬───────────────────────────────┬──────────────┘
                │                               │
                │ bounded workflow              │ execution-scoped MCP
                ▼                               ▼
┌────────────────────────────┐      ┌───────────────────────────┐
│ Agentplane task            │      │ Nimbalyst                 │
│ WorkOrder / result / ACR   │      │ visible workstream        │
│ verification / findings    │      │ sessions + transcripts    │
└────────────────────────────┘      │ Claude / Codex / others   │
                                    └────────────┬──────────────┘
                                                 │
                                                 ▼
                                    ┌───────────────────────────┐
                                    │ Git worktree / branch     │
                                    └───────────────────────────┘
```

## 5. Source-of-truth matrix

| Question | Authoritative source |
|---|---|
| What work exists? | GitHub Issue |
| What is highest priority? | GitHub Project fields |
| Which specialized team is eligible? | GitHub Project `Team/Area` + policy config |
| Is work blocked? | GitHub dependencies/sub-issues + Project state |
| Who locally owns the attempt now? | `acpd` SQLite lease |
| Which execution/session belongs to it? | `acpd` run/session links |
| What workflow step is next? | Agentplane task state |
| What did the provider actually emit? | Nimbalyst raw transcript |
| What changed in the repository? | Git/worktree observation |
| What checks were observed? | Agentplane receipt / `acpd` observed artifact |
| What is shown to the human? | Nimbalyst + control panel read model |
| What is committed as history? | sanitized closeout and ACR only |

No field may have two writable owners.

## 6. GitHub issue model

### Required existing fields retained

The current skill already expects:

- Milestone;
- Priority;
- Phase;
- Status;
- Iteration for active statuses.

These stay.

### Added routing/runtime projection fields

Recommended GitHub Project fields:

| Field | Values / purpose |
|---|---|
| `Team` | backend, frontend, data, release, security, research, general |
| `Agent Profile` | planner, implementer, reviewer, investigator, operator |
| `Automation` | manual, assisted, auto |
| `Agent State` | unclaimed, claiming, running, waiting-human, review, failed |
| `Run ID` | current local run correlation id |
| `Risk` | normal, elevated, critical |
| `Human Gate` | none, plan, publish, merge, deploy |

These fields are projection/coordination hints. Local process truth never comes from `Agent State` alone.

### Normalized issue

```ts
interface NormalizedIssue {
  id: string;
  number: number;
  repo: string;
  title: string;
  body: string;
  url: string;
  priority: number | null;
  phase: string | null;
  status: string;
  iteration: string | null;
  team: string | null;
  agentProfile: string | null;
  automation: 'manual' | 'assisted' | 'auto';
  risk: 'normal' | 'elevated' | 'critical';
  labels: string[];
  assignees: string[];
  blockedBy: Array<{ id?: string; number?: number; state?: string }>;
  dispatchable: boolean;
  dispatchReasons: string[];
  updatedAt: string;
}
```

### Eligibility

An issue is dispatchable only when all conditions hold:

1. board admission is `PASS`;
2. status belongs to configured active states;
3. required fields exist;
4. all blockers are terminal/accepted;
5. automation is not `manual` unless explicitly started;
6. team/profile mapping exists;
7. no active local lease exists;
8. global and team concurrency slots exist;
9. no unresolved human gate blocks the next step;
10. latest issue revision still matches the prepared claim.

### Ordering

Default deterministic ordering:

```text
Priority ascending
→ explicit owner/team match
→ active iteration
→ oldest ready_at
→ issue number
```

The LLM does not select the next issue. It receives the issue selected by scheduler code.

## 7. Runtime database

### Boundary

SQLite stores **structured runtime coordination only**. It does not duplicate Nimbalyst's full provider transcript.

Recommended local path:

```text
%LOCALAPPDATA%/agent-control-plane/control.db   Windows
~/.local/share/agent-control-plane/control.db  Linux
~/Library/Application Support/agent-control-plane/control.db  macOS
```

One daemon process owns all writes.

### Core tables

```text
repositories
project_bindings
issue_snapshots
claims
runs
session_links
worktree_links
runtime_events
artifacts
checkpoints
outbox
reconciliation_cursor
```

### Minimal schema responsibilities

#### `claims`

```text
issue_id              unique while active
run_id
owner_instance_id
lease_expires_at
fencing_token
issue_revision
state                  claiming | active | releasing | terminal
```

#### `runs`

```text
run_id
issue_id
agentplane_task_id
state
team
profile
started_at
ended_at
stop_reason
closeout_state
```

#### `session_links`

```text
run_id
nimbalyst_session_id
provider
provider_session_id
role
parent_session_id
worktree_id
started_at
ended_at
status
```

#### `runtime_events`

Append-only normalized facts:

```text
sequence
run_id
session_id nullable
event_type
source
occurred_at
payload_json
payload_hash
```

#### `outbox`

All GitHub/Agentplane/Nimbalyst side effects are persisted before execution:

```text
idempotency_key
operation_type
payload
state
attempts
last_error
next_attempt_at
```

### Transaction example

Claiming an issue is one SQLite transaction:

```text
verify no active claim
insert claim + fencing token
insert run
append run.claimed
enqueue GitHub projection update
commit
```

GitHub update is eventually consistent through outbox and mandatory read-back.

## 8. Nimbalyst operating model

## 8.1. Visible orchestrator

For the first usable version, each repository has one visible Nimbalyst session:

```text
ACP Orchestrator — <repository>
```

It uses:

- `control.next_assignment` from our MCP;
- `nimbalyst-host.spawn_session`;
- `get_session_status` / `get_session_result`;
- `respond_to_prompt`;
- `notify_user`;
- workstream overview.

The orchestrator is visible in the same interface as its children. It does not maintain a hidden PowerShell runner.

### Authority boundary

The orchestrator session may:

- request the deterministic next assignment;
- create the requested child session;
- inspect child result;
- submit typed semantic result/handoff;
- ask the user.

It may not:

- choose a lower-priority issue itself;
- manufacture a claim;
- mark observed tests green;
- mutate another run directly;
- close an issue without supervisor acceptance.

## 8.2. Team representation

A team is a policy/template, not a permanent process collection.

```yaml
team: standard-feature
max_parallel: 2
roles:
  planner:
    provider_policy: reasoning-strong
    tools: read
  implementer:
    provider_policy: coding-strong
    tools: full
  reviewer:
    provider_policy: independent-review
    tools: read
policies:
  reviewer_must_differ_from_implementer: true
  plan_requires_human_for_risk: elevated
  merge_requires_human: true
```

Sessions exist only while a run needs them.

## 8.3. Workstream mapping

```text
GitHub Issue #N
  ↕ one ACP run
Nimbalyst workstream
  ├── planner session
  ├── implementer session
  ├── tester session optional
  └── reviewer session
```

Each child session is linked in `session_links` immediately after creation.

## 9. Agentplane integration

### Ownership

Agentplane is the authoritative per-issue workflow state only after a task has been materialized for the issue. It must not independently poll and claim GitHub Issues in the MVP.

### Mapping

```text
GitHub issue ID       → one Agentplane task ID
Agentplane episode    → one Nimbalyst child session or bounded turn
Semantic result       → returned to Agentplane
Execution receipt     → supervisor-observed evidence
ACR                    → closeout artifact
```

### Initial integration style

Use Agentplane's external-agent route:

1. bridge asks Agentplane for next typed step;
2. if `agent_episode`, create Nimbalyst session with WorkOrder;
3. child writes typed result through execution-scoped MCP/result path;
4. bridge returns result to Agentplane;
5. Agentplane advances or stops at approval/wait/terminal;
6. `acpd` mirrors only the high-level step/event.

This avoids nested managed runners during MVP.

### Fallback if the spike fails

If Agentplane external integration proves too brittle, reuse its public contract ideas and schemas, but implement only this subset locally:

- WorkOrder;
- SemanticResult;
- StateFingerprint;
- ExecutionReceipt;
- ACR.

A full custom workflow engine remains out of scope.

## 10. MCP surface

Agent sessions access one execution-scoped MCP server.

### Read tools

```text
control.get_assignment
control.get_run_context
control.get_issue_snapshot
control.get_dependencies
control.get_artifacts
control.get_handoff_context
```

### Write tools

```text
control.report_progress
control.record_decision
control.publish_artifact
control.request_human_input
control.propose_handoff
control.submit_semantic_result
control.report_failure
```

### Forbidden tools

The MCP does not expose:

```text
claim_next_issue
spawn_raw_process
edit_registry
set_other_session_state
write_delivery_log
mark_tests_verified
force_close_issue
```

The token is scoped to one run/session/role and validated against fencing token.

## 11. Automatic preservation and burial

This is a first-class subsystem, not an instruction to agents.

## 11.1. Raw session record

Nimbalyst automatically persists:

- user/agent messages;
- provider-native events;
- tool calls/results;
- nested subagents;
- interactive requests;
- session/provider IDs;
- edited files and worktree linkage where available.

The agent is not responsible for saving its own transcript.

## 11.2. Structured checkpoint

A checkpoint is generated by an observer at these boundaries:

- planning result accepted;
- child session becomes idle/waiting/terminal;
- human question created;
- verification completed;
- provider error;
- daemon shutdown/restart reconciliation;
- periodic maximum event interval.

Checkpoint payload:

```json
{
  "run_id": "...",
  "issue": "owner/repo#123",
  "sessions": ["..."],
  "state": "running",
  "current_step": "implementation",
  "last_activity": "...",
  "edited_files": [],
  "artifacts": [],
  "pending_questions": [],
  "agentplane_fingerprint": "...",
  "git_head": "..."
}
```

It is derived from host/supervisor state. Missing agent prose cannot block it.

## 11.3. Run bundle

On terminal/abandoned run the archive coordinator creates:

```text
<local-data>/archives/<repo>/<issue>/<run-id>/
├── manifest.json
├── timeline.jsonl
├── session-links.json
├── artifacts.json
├── decisions.jsonl
├── questions.jsonl
├── git-state.json
├── checks.json
├── transcript-export/
│   ├── session-<id>.jsonl
│   └── session-<id>.md
└── closeout.md
```

### Git boundary

Raw transcript bundle stays host-local and may contain sensitive content.

Only sanitized artifacts may be projected into the project repository:

```text
.agent-control-plane/evidence/<issue>/<run-id>/
├── README.md
├── acr.json
├── artifacts.json
└── checks.json
```

Secret scan/redaction runs before projection.

## 11.4. Closeout dispositions

Every unfinished item receives exactly one disposition:

```text
DONE
CANCELLED
SUPERSEDED
FOLLOW_UP_ISSUE:<number>
HANDED_OFF:<run-or-team>
BLOCKED_EXTERNAL
NEEDS_OWNER
FAILED_RETRYABLE
FAILED_TERMINAL
```

No session/worktree may be deleted until closeout and artifact references are durable.

## 12. Operator interface

### MVP interface

Use existing surfaces:

- GitHub Project — work board;
- Nimbalyst Agent Manager — sessions/workstreams/transcript;
- CLI `acp status` — joined runtime view.

This is enough to start using the flow without building an application.

### Later Nimbalyst extension panel

Panel `Issue Control` shows:

- dispatchable issue queue;
- current claim/run;
- linked planner/implementer/reviewer sessions;
- provider/model/worktree;
- last activity;
- pending question;
- edited files/checks/artifacts;
- Agentplane step and approval;
- retry/stop/answer/release controls.

The panel reads the daemon API, not Nimbalyst internal DB schema. It may use host-provided session IDs only as links.

## 13. Reconciliation

`acpd` never relies only on notifications.

Each cycle compares:

1. latest GitHub issue/project snapshot;
2. active SQLite claim/run;
3. Agentplane task/step;
4. Nimbalyst session status;
5. Git worktree/process facts;
6. pending outbox.

Example rules:

```text
GitHub issue no longer active + run active
  → stop new episodes, request safe cancellation/closeout

SQLite run active + Nimbalyst session missing
  → classify session_lost; resume/new session or needs-human

Nimbalyst session complete + Agentplane waiting for result
  → collect result and resume workflow

Agentplane terminal + GitHub still running
  → enqueue verified status projection

worktree residual after terminal
  → block cleanup and expose incident
```

## 14. Security

1. Use a dedicated GitHub App/automation identity, not owner credentials.
2. Child provider processes receive an environment allowlist.
3. Raw transcript never auto-commits.
4. Every MCP token is scoped to repository, run, session and role.
5. All commands validate workspace root and reject traversal/symlinks.
6. Human approvals are bound to issue revision, plan digest and run fingerprint.
7. Secrets are stored in OS credential storage, not SQLite payloads or logs.
8. Redaction occurs before summary/ACR/GitHub comment generation.
9. Elevated/critical issues cannot auto-publish or merge.
10. Old process writes are rejected by fencing token after lease generation changes.

## 15. Invariants

1. One issue has at most one active local run.
2. One worktree has at most one active writing run.
3. One run has one current fencing token.
4. A session cannot submit state for another run.
5. Agent claims never become observed evidence automatically.
6. GitHub field update is not considered complete before read-back.
7. Session deletion requires durable closeout.
8. Worktree deletion requires captured Git/evidence state.
9. Active run must be visible through `acp status` even if Nimbalyst UI is closed.
10. Missing provider telemetry is `unavailable`, never guessed.

## 16. Non-goals for MVP

- multi-host/distributed execution;
- remote A2A agents;
- custom desktop/mobile clients;
- replacing GitHub Projects;
- replacing Nimbalyst transcript storage;
- hosting models;
- general-purpose workflow designer;
- autonomous merge/deploy;
- exactly-once side effects;
- long-lived permanent worker roles;
- storing private chain-of-thought.

## 17. Consequences

### Positive

- useful MVP without building a new UI/runtime;
- existing Agent Team planning knowledge survives;
- complete session history no longer depends on agent discipline;
- different providers remain usable;
- issue-first dynamic scheduling becomes deterministic;
- human can observe both orchestrator and children;
- evidence is separated from claims.

### Negative

- four explicit systems must be linked carefully;
- Nimbalyst internal meta-agent APIs may evolve;
- Agentplane integration adds a second task artifact, even though ownership is bounded;
- GitHub/local runtime is eventually consistent;
- a small daemon and extension still need maintenance.

These costs are lower than maintaining another full agent application.

## 18. Review trigger

Revisit this ADR only if a validation spike proves one of the following:

- Nimbalyst cannot expose/persist child sessions reliably;
- Agentplane cannot be driven as external workflow without manual choreography;
- GitHub Project API cannot support required read-back/claim projection;
- the extension boundary cannot link sessions safely;
- one of the selected projects changes license or is abandoned without a maintainable fork path.
