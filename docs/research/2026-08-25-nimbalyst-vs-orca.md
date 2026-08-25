# Nimbalyst vs Orca for Agent Team / Oil Automation

**Date:** 2026-08-25

## Executive verdict

Orca is materially closer to the original Agent Team problem than the previous research assumed.

Nimbalyst is a strong visual workbench/session host. Orca is also a workbench, but it additionally ships an **experimental structured orchestration runtime** with durable Runs, Tasks, Dispatches, Messages, heartbeats and decision gates. It also has native GitHub Issues / PR / Actions / Projects surfaces and treats the real agent terminal as the source of truth.

For the current Oil Automation workflow, Orca should be tested **before committing to Nimbalyst as the default host**.

Recommended evaluation order:

```text
1. Orca pilot on one standard GitHub issue
2. Nimbalyst pilot on the same workflow
3. choose the operator/runtime host
4. keep Agentplane selective for critical evidence/governance only
5. do not build acpd until both products fail a concrete required invariant
```

## 1. Why Orca changes the picture

Orca is not only a parallel-terminal/worktree manager.

Its `orchestration` skill exposes this model:

```text
Run
  durable namespace + coordinator inbox

Task
  spec + dependencies + status
  pending | ready | dispatched | completed | failed | blocked

Dispatch
  one attempt of one Task on one worker terminal
  lifecycle authority for worker_done / heartbeat

Message
  persistent coordination mail
  status | dispatch | worker_done | escalation | question | heartbeat | ...

Decision gate
  coordinator-owned blocking decision attached to a Task
```

The preferred supervised loop is:

```text
run-create
→ task-create
→ worker-start
→ check --wait
→ worker_done / question / escalation
→ release / retry / next wave
```

Workers receive `taskId` and `dispatchId`; completion authority belongs to the active Dispatch, and stale retries cannot legitimately complete a different dispatch. Tasks can form a DAG. Workers can send heartbeats and blocking questions. A coordinator can create decision gates. Orca can also start workers on connected remote Orca servers.

This is structurally very close to what Agent Team evolved toward, but the state is owned by the application runtime/database instead of Markdown inboxes and PowerShell processes.

## 2. Direct mapping from legacy Agent Team

| Legacy Agent Team | Orca |
|---|---|
| team/sprint namespace | Run |
| slice/task in inbox | Task |
| one worker attempt | Dispatch |
| message MD file | Message / Delivery |
| `status: unread/done` | runtime mailbox acknowledgement + task/dispatch state |
| watcher | runtime events / terminal hooks / `check --wait` |
| headless runner | supervised worker terminal |
| `delivery_id` | message/delivery/dispatch IDs |
| retry | explicit `worker-start --retry-of` |
| heartbeat/liveness | Dispatch heartbeat + agent state hooks |
| question to orchestrator | `orchestration ask` |
| human/decision gate | decision gate |
| audit of worker terminal | `worker-read`, persistent terminal/session history |
| parallel roles | multiple Tasks/Workers |
| dependency graph | Task DAG |
| cross-host worker | federated worker `--on <environment>` |

The biggest remaining difference is methodology: Orca does not know the Oil-specific planning, independent-review, file-zone, TDD and risk rules unless a skill/policy tells the coordinator how to use the primitives.

## 3. Nimbalyst vs Orca

| Dimension | Nimbalyst | Orca | For Oil |
|---|---|---|---|
| Native Windows | Yes | Yes, signed Windows builds | Tie |
| Claude Code | Deep | Deep | Tie |
| Codex | Deep, including App Server/ACP parsing | Deep CLI integration + account/session hooks | Tie |
| Other CLI agents | Several, extensible | Very broad: effectively any CLI agent | Orca |
| Raw terminal transparency | Claude CLI raw drawer exists; structured transcript is central | Terminal is primary/source-of-truth for all CLI agents | **Orca** |
| Structured transcript | Excellent normalized transcript pipeline | Experimental Chat UI over PTY | Nimbalyst |
| Tool-call visibility | Strong | Terminal + structured Chat UI/ACP where supported | Tie / Nimbalyst slight |
| Session history | Own raw provider log + search | Reads provider-native histories; AI Vault; raw log path/resume | Different strengths |
| Terminal scrollback restart | Not the central persistence model | Explicitly survives restart | Orca |
| Parallel worktrees | Yes | Core product model | Tie / Orca slight |
| Visual diff/review | Strong | Strong + annotations | Tie |
| GitHub Issues | Import/adoption/overlay into tracker | Native issue drawer linked directly to worktrees | **Orca** |
| GitHub Projects | Not found as an equivalent canonical ProjectV2 operator surface | Full Projects view under Tasks, create worktree from card | **Orca** |
| GitHub PR/Actions | Available through normal Git/workflows but not core differentiator | First-class PR/check/comments/Actions UI | **Orca** |
| Local tracker | Very rich database-first tracker | Tasks primarily integrate GitHub/Linear/Jira + workspace statuses | Nimbalyst |
| Multi-agent structured runtime | Cross-session MCP/meta-agent tools | **Run/Task/Dispatch/Message/Gate runtime** | **Orca** |
| Task DAG | Can be modeled through tracker/workstreams, not core runtime contract | First-class orchestration Task deps | **Orca** |
| Durable coordinator inbox | Cross-session host operations | First-class Run inbox/Delivery | **Orca** |
| Heartbeats / worker completion authority | Session state/result APIs | Explicit Dispatch authority + heartbeat + `worker_done` | **Orca** |
| Remote execution | Collaboration/mobile; provider/runtime features | SSH, remote Orca server, per-workspace VM, federated workers | **Orca** |
| Mobile | Strong companion | Strong iOS/Android companion | Tie |
| Visual documents | Markdown, Mermaid, Excalidraw, CSV, data models, extensible editors | Editor, Markdown, Mermaid/PDF/images, browser Design Mode | **Nimbalyst** |
| Extension model | Mature panels/editor/AI tools | Plugins/skills/MCP, rapidly evolving | Nimbalyst currently richer for visual extension |
| Community/adoption | ~1.5k stars in researched snapshot | tens of thousands of stars; thousands of forks; very active | **Orca** |
| Orchestration maturity | Cross-session runtime is real, but not a formal Task/Dispatch protocol | Orchestration exists but is explicitly **experimental** | Orca capability, but must validate |

## 4. Observability: Orca directly answers the main user complaint

The original Agent Team was created primarily for observability. Orca is terminal-first:

- every agent is a real terminal session;
- working / waiting / done / failed state is visible on cards;
- terminal output and commands remain visible;
- scrollback is searchable and survives restart;
- the Agents feed shows completions, blockers and response previews across worktrees;
- clicking an event jumps to the exact agent terminal;
- optional Chat UI is a structured projection over the same PTY, not a separate hidden execution path.

This is a better match for the requirement “I want to see what the orchestrator/worker is actually doing” than a chat-first UI where execution may be represented as a spinner.

Nimbalyst is stronger if the desired primary representation is a normalized, searchable transcript with rich tool cards and visual artifacts.

## 5. GitHub Projects may require no custom bridge with Orca

Orca already exposes a full GitHub Projects view under **Tasks**. It can:

- browse project cards across repositories;
- filter by source repo;
- open GitHub issue history/activity;
- create a worktree from an issue/project card;
- keep the linked issue/PR attached to that worktree;
- show PR state, comments, reviews, Actions/check failures and logs inline.

This is much closer to the existing Oil workflow than Nimbalyst's current GitHub issue import/adoption model.

A plausible minimal Oil flow becomes:

```text
GitHub Project card
→ Create Orca workspace/worktree
→ coordinator starts an Orca Run
→ Tasks: planner / implementer / reviewer
→ worker-start for each role
→ visible raw terminals + structured messages
→ PR / Actions remain inside Orca/GitHub
→ Run completes
```

No custom SQLite scheduler is needed for this human-started flow.

## 6. What Orca still does NOT replace

### 6.1. Oil methodology

Orca primitives do not automatically enforce:

- roadmap → plan → feature decomposition;
- file-zone conflict policy;
- independent reviewer != implementer;
- TDD RED→GREEN;
- domain-specific verification;
- risk-tiered human approval;
- closeout dispositions/follow-up issue rules.

The legacy skill should still be reduced into an **Orca-native methodology skill** that creates the right Run/Task DAG and applies these policies.

### 6.2. Agentplane's evidence/governance boundary

Orca orchestration gives stronger coordination than Nimbalyst, but it is not the same as Agentplane.

Agentplane still provides concepts that Orca's public orchestration contract does not currently match directly:

- StateFingerprint-bound semantic results;
- explicit separation of agent-claimed checks vs supervisor-observed verification receipts;
- bounded WorkOrder authority/writable scope;
- effect intent and `effect_in_doubt` reconciliation;
- formal ACR closeout.

Therefore Agentplane remains useful for Tier C work (finance formulas, security/identity, production/deploy, irreversible migrations), but Orca likely removes the need to use Agentplane as the general multi-agent coordinator.

## 7. Important Orca risks

### Experimental orchestration

The structured Run/Task/Dispatch layer is explicitly marked experimental. Command contracts have already migrated: legacy scheduler commands were retired in favor of Run + worker-start. A pilot must include crash/restart, stale-dispatch, duplicate completion and mailbox replay tests.

### Permission defaults are too permissive for Oil

Orca launches supported agents with full-autonomy/bypass flags by default (for example Claude `--dangerously-skip-permissions` and Codex sandbox/approval bypass). The product rationale is that the git worktree acts as a disposable engineering sandbox.

For Oil this default is **not acceptable** for critical work: a worktree does not isolate network credentials, production services, databases or host filesystem outside the checkout.

Required policy for pilot:

- set agent permissions to Manual or explicit restricted args;
- isolate credentials;
- keep human merge/deploy gates;
- use SSH/remote per-workspace VM/container-style environments for higher-risk executions where appropriate;
- never assume worktree isolation == security sandbox.

### Transcript persistence model differs from Nimbalyst

Nimbalyst persists a normalized provider-native raw log in its own database. Orca primarily reads the transcript stores written by the underlying agent CLIs and preserves PTY scrollback/session references. This is very transparent and portable, but a future provider that does not leave a usable session log may have weaker archival behavior.

## 8. Revised shortlist

For the exact Oil/Agent Team use case:

### #1 Orca — strongest candidate to pilot first

Why:

- native Windows;
- terminal-first observability;
- GitHub Projects/Issues/PR/Actions built in;
- real structured multi-agent orchestration;
- worktrees;
- broad provider support;
- remote/federated execution;
- large/active open-source ecosystem.

### #2 Nimbalyst — strongest visual/session workspace

Prefer if:

- normalized transcript/archive is more important than raw terminal;
- visual docs/Excalidraw/data models are part of the daily workflow;
- Nimbalyst's cross-session MCP is enough and a formal Task/Dispatch DAG is not needed;
- richer visual extension panels are more important than native GitHub Projects integration.

### #3 Agentplane — selective quality/evidence layer

Not an alternative to Orca/Nimbalyst UI. Pilot on Tier C issues after choosing the host.

## 9. New recommended pilot

Do the same real issue in both products rather than building anything:

### Orca scenario

```text
GitHub Issue
→ worktree from Tasks/Project card
→ Run
→ planner Task
→ implementer Task
→ reviewer Task
→ decision gate when needed
→ PR/Actions
```

Verify:

- Run/Task/Dispatch survive Orca restart;
- terminal history remains inspectable;
- messages replay until acknowledged;
- duplicate `worker_done` does not incorrectly settle a retry;
- parent can read worker output after release;
- Codex and Claude workers behave symmetrically enough;
- GitHub Project linkage remains intact;
- permission defaults can be made acceptable on Windows.

### Nimbalyst scenario

Use the existing planner → implementer → reviewer workstream test and compare:

- visibility;
- handoff friction;
- archival quality;
- GitHub integration;
- review experience;
- restart recovery;
- amount of custom policy glue needed.

## 10. Bottom line

Before this comparison, the minimal recommendation was:

```text
GitHub + Nimbalyst + small Agent Team methodology
```

After reviewing Orca's current orchestration and GitHub surfaces, the recommendation becomes:

```text
GitHub + [Orca OR Nimbalyst]
            ↑
       choose by pilot

Agent Team methodology → native skill for chosen host
Agentplane             → only for critical evidence/governance
Custom acpd            → deferred
```

For the user's current priorities — **observable multi-agent coordination, GitHub Projects, raw execution visibility, Windows, and minimal custom development** — Orca is presently the stronger first candidate.