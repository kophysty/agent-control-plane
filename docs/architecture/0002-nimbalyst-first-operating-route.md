# ADR-0002 — Nimbalyst-first operating route; GitHub retained, Agentplane selective

- **Status:** Recommended for immediate pilot; owner confirmation pending
- **Date:** 2026-08-25
- **Supersedes:** not ADR-0001 as a long-term architecture, but narrows its immediate implementation route
- **Decision objective:** start using visible agent teams now without first building `acpd`, a custom UI, or a new runtime

## 1. Decision

For the current Oil Automation scale, use this stack first:

```text
GitHub Issues + Projects
  = canonical business backlog, priority, acceptance, PR/CI linkage

Nimbalyst
  = operator cockpit, workstreams, sessions, transcripts, tools, worktrees

Agent Team methodology
  = a much smaller Nimbalyst-native policy/role skill

Agentplane
  = optional per-task governance/evidence layer for critical work only

Custom Agent Control Plane daemon
  = deferred until manual operation proves a concrete repetitive need
```

The immediate target is **not** autonomous issue polling and dispatch. The immediate target is a human-visible planner → implementer → reviewer workflow whose sessions and actions are preserved automatically.

## 2. What Nimbalyst replaces

Nimbalyst can replace most runtime responsibilities of the legacy Agent Team Skill:

- visible Claude Code and Codex sessions;
- parent/child session orchestration;
- `spawn_session`, `send_prompt`, `get_session_status`, `get_session_result`;
- workstreams and optional worktrees;
- persisted provider-native transcripts;
- tool calls/results and edited-file linkage;
- subagent representation;
- questions/permissions and notifications;
- session Kanban;
- terminal and visual diff surfaces.

The current Nimbalyst source also contains:

- append-only raw provider messages as transcript truth;
- a collapsible extended-thinking renderer when a provider emits thinking;
- user-facing tool-call visibility enabled by default;
- a genuine, resizable **Raw terminal** drawer for `claude-code-cli` sessions;
- GitHub issue overlay/adoption into the local tracker.

Therefore file inboxes, watchers, headless inbox runners, role PIDs, NNN allocation, `delivery.log` and manual transcript checkpoints should not be rebuilt.

## 3. What Nimbalyst does not replace

Nimbalyst is primarily a session/workbench host. It does not by itself provide all of the following guarantees:

- GitHub ProjectV2 priority/iteration/milestone truth;
- server-side PR/CI history and branch protection;
- atomic issue claim across multiple independent supervisors;
- stale-result rejection bound to Git/task state;
- formal separation between agent-claimed tests and supervisor-observed checks;
- effect-in-doubt recovery for ambiguous external side effects;
- guaranteed structured closeout/ACR for every run;
- deterministic portfolio-level issue scheduling.

Its session Kanban organizes sessions/workstreams through phases such as backlog, planning, implementing, validating and complete. It is an execution view, not automatically the canonical project backlog.

Its tracker is database-first and capable, but GitHub adoption is explicitly a one-way escalation/overlay path. Making both GitHub and the Nimbalyst tracker writable task authorities would reintroduce two sources of truth.

## 4. GitHub decision

**Do not abandon GitHub for Oil Automation.**

Observed project scale on 2026-08-25:

- 577 issues in repository history;
- 241 open issues;
- 1,021 pull requests;
- a live Project board with Current iteration, Roadmap, History and Backlog views;
- CI, production deploy, live smoke and drift workflows;
- issue bodies ranging from compact spec references to detailed problem/goal/scope/acceptance records;
- a PR template enforcing spec, RED→GREEN, independent review, smoke/evidence and cleanup gates.

This is already beyond the threshold where a local-only tracker is an adequate sole source of truth.

### What is good in the current GitHub layer

- durable problem and acceptance history;
- issue ↔ PR ↔ commit ↔ CI linkage;
- milestone/iteration/priority separation;
- explicit dependencies and spec links;
- server-side review and protection boundary;
- a board that can outlive any local workstation or Nimbalyst database.

### What needs simplification

The current Board screenshot shows approximately:

```text
Todo         237
Spec           0
In progress    3
Review         2
In Main        0
```

This suggests that the main board mixes the long-term backlog with ready work, while `Spec` and `In Main` may not be earning their operational cost. Active columns also display zero aggregate estimate despite issue-size markers, suggesting inconsistent field upkeep.

Recommended changes:

1. Keep GitHub as the canonical work tracker.
2. Separate `Backlog` and `Ready`; do not put hundreds of unscheduled items in one operational `Todo` column.
3. Treat specification as an admission property/gate, not necessarily a permanent status column.
4. Keep hard fields minimal:
   - Status;
   - Priority;
   - Milestone or active Iteration when scheduled;
   - Area/Team;
   - Risk for critical work.
5. Keep Size/Estimate advisory unless the values are actually used for planning.
6. Apply process gates by risk profile instead of forcing every PR through every expensive gate.
7. Do not add live `Agent State`, lease or PID truth to GitHub until an actual scheduler exists.
8. Link the GitHub issue to its Nimbalyst workstream/session, but keep runtime status local to Nimbalyst.

## 5. Agentplane explained

Agentplane is not a backlog, IDE, chat or multi-agent dashboard.

It is a **deterministic quality/lifecycle controller around one engineering task**.

Concrete mapping:

```text
GitHub Issue
  says WHAT must be done and why it matters

Nimbalyst
  shows WHO is working, their chats, tools, terminals and files

Agentplane
  decides WHICH controlled step is legal next and WHAT evidence is required
```

For one issue Agentplane can:

1. issue a bounded PLANNER WorkOrder;
2. persist the returned plan;
3. stop for real human approval;
4. issue an EXECUTOR WorkOrder with exact checkout/write scope and stop rules;
5. reject a result if task/Git/provider state changed since preparation;
6. independently observe diff, files, commands and checks;
7. run an independent EVALUATOR episode;
8. request bounded rework;
9. manage branch/PR/verification/integration boundaries;
10. produce an Agent Change Record and typed terminal state.

Its most valuable distinction is:

```text
agent says tests passed
!=
supervisor observed tests passed
```

Agentplane works without GitHub through its local task backend, but combining Nimbalyst Tracker and Agentplane local tasks would still create two task stores. For Oil, GitHub should remain the backlog; Agentplane, if adopted, should own only the execution/evidence contract of selected high-risk issues.

## 6. Risk-tiered operating model

### Tier A — lightweight

Use for docs, research, local investigation and trivial reversible fixes.

```text
GitHub issue or direct request
→ one visible Nimbalyst session
→ normal tests/diff
→ PR when repository history is needed
```

No Agentplane.

### Tier B — standard engineering change

```text
GitHub issue
→ Nimbalyst workstream
   ├── planner
   ├── implementer
   └── independent reviewer
→ existing PR template + CI
```

No custom daemon; Agentplane optional.

### Tier C — critical

Use for production deployment, security/identity, finance/formulas, irreversible migrations and customer-visible calculations.

```text
GitHub issue
→ Agentplane task/WorkOrders
→ Nimbalyst planner/implementer/evaluator sessions
→ observed verification receipt
→ human publish/merge gate
→ ACR/closeout
```

Agentplane is piloted here first.

## 7. Revised roadmap

### P0 — operate Nimbalyst manually first

Duration target: several real working days, not a synthetic demo only.

Verify on Windows:

- Claude Code and Codex session persistence after app restart;
- tool calls and edited files remain visible;
- child sessions can be spawned and resumed;
- parent can obtain child status/result;
- questions and permissions are visible;
- a dedicated worktree can be used safely;
- extended-thinking blocks are visible when supplied;
- Claude CLI Raw terminal works and is usable;
- exact Codex transparency/terminal behavior is documented.

### P1 — shrink Agent Team into a Nimbalyst-native methodology

Retain:

- roadmap/plan/feature decomposition;
- dependencies and file zones;
- planner/implementer/reviewer roles;
- independent review;
- human gates;
- structured decisions and handoff briefs.

Remove:

- `.agent-team` transport;
- inbox/outbox;
- watcher/runner;
- registry liveness;
- delivery logs;
- manual checkpoint obligations;
- direct provider process ownership.

### P2 — selective Agentplane spike

Run one real Tier C issue through:

```text
planner → approval → executor → observed checks → evaluator → closeout
```

Adopt only if the evidence/recovery value exceeds the integration friction.

### P3 — defer automation bridge

Do not build SQLite claims, GitHub polling or an `acpd` daemon until real operation shows at least one of these recurring failures:

- two sessions race for the same issue;
- manual priority selection consumes material time;
- status/read-back drift recurs;
- automatic closeout is repeatedly missing;
- more than one host needs to claim work;
- unattended dispatch becomes an explicit requirement.

## 8. Alternatives

### Agent of Empires

Strongest alternative when raw terminal transparency, tmux persistence, web/mobile terminal and broad CLI support dominate. However native Windows is not supported; it requires WSL2 and POSIX/tmux process handling. It is also weaker than Nimbalyst in visual documents, local tracker/workstream richness and Windows-native desktop UX.

### Kandev

Interesting when Docker/SSH/cloud executors and distributed work are current requirements. This is not the present Oil bottleneck.

### OpenHands

A much larger autonomous coding platform, but would replace the execution model rather than simply host existing Claude Code/Codex workflows. Too large a change for the immediate goal.

### Claude Squad / dmux

Good lightweight terminal/worktree multiplexers. They do not solve persistent rich transcripts, tracker linkage and visual operator workflow as fully as Nimbalyst.

## 9. Final recommendation

The correct immediate route is:

```text
KEEP GitHub Projects
ADOPT Nimbalyst as the replacement for Agent Team runtime/UI
REDUCE the custom skill to methodology and Nimbalyst orchestration
PILOT Agentplane only for critical work
DO NOT build acpd or a new application yet
```

Nimbalyst alone is sufficient to test whether visible session-based teams solve the day-to-day pain. GitHub remains necessary at Oil scale. Agentplane is a selective quality/evidence multiplier, not a mandatory foundation for every issue.
