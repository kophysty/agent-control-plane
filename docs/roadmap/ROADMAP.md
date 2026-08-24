# Agent Control Plane Roadmap

**Стратегия:** validation first, integration before invention.  
**Цель:** начать реально пользоваться управляемыми агентскими командами до разработки собственного UI/runtime.

## 0. Release gates

Ни один следующий этап не начинается, пока предыдущий не оставил воспроизводимый evidence.

```text
R0  Existing products actually satisfy the assumed capabilities
R1  One issue can be claimed and linked to one visible workstream
R2  One full planner → implementer → reviewer flow is durable
R3  Crash/restart cannot lose or duplicate the run
R4  Automatic closeout works without agent remembering it
R5  Only then build convenience UI and broader automation
```

## Milestone V0 — Freeze and baseline

### Goal

Остановить дальнейшее расширение файлового Agent Team runtime и зафиксировать migration boundary.

### Work

- [ ] Пометить `Skill/agent-team` v9 как legacy runtime.
- [ ] Запретить новые watcher/runner/inbox semantics.
- [ ] Сохранить все validation suites как specification fixtures.
- [ ] Составить migration inventory по компонентам:
  - carry as policy;
  - port to daemon;
  - replace with Nimbalyst;
  - replace with Agentplane;
  - archive/delete.
- [ ] Создать sanitized export legacy teams/messages для будущей reconciliation.
- [ ] Не массово менять `unread` и не удалять inbox.

### Acceptance

- Есть одна таблица migration disposition для каждого script/reference/widget component.
- Новая работа не добавляется в lifecycle-v9 scope.

### STOP

Если v9 продолжает получать новые runtime-фичи параллельно, новая архитектура не стартует: две системы будут менять один и тот же workflow.

---

## Milestone V1 — Nimbalyst black-box validation

### Goal

Доказать, что Nimbalyst действительно может заменить chat/session/watcher/widget слой.

### V1.1 Claude session durability

- [ ] Создать Nimbalyst workspace в test repository.
- [ ] Запустить Claude Code session.
- [ ] Выполнить tool calls, edit и child subagent.
- [ ] Закрыть/restart Nimbalyst.
- [ ] Открыть session и проверить:
  - user/assistant messages;
  - tool calls/results;
  - nested subagent transcript;
  - edited files;
  - provider session id;
  - worktree link.

### V1.2 Codex session durability

Повторить тот же сценарий для Codex.

### V1.3 Cross-session orchestration

В видимой orchestrator session:

- [ ] вызвать `spawn_session` для Claude child;
- [ ] вызвать `spawn_session` для Codex child;
- [ ] получить `get_session_status`;
- [ ] отправить follow-up через `send_prompt`;
- [ ] получить `get_session_result`;
- [ ] ответить на permission/question через `respond_to_prompt`;
- [ ] проверить `notifyOnComplete` и `notify_user`;
- [ ] проверить отдельный worktree.

### V1.4 Automatic observer

- [ ] Убедиться, что transcript сохраняется host-ом даже когда child не пишет checkpoint.
- [ ] Зафиксировать стабильные IDs и доступные metadata.
- [ ] Проверить, какие lifecycle events доступны extension/host surface.

### Evidence

```text
evidence/v1-nimbalyst/
├── environment.md
├── claude-session.md
├── codex-session.md
├── cross-session.md
├── restart-results.md
└── capability-matrix.json
```

### Acceptance

- Оба providers переживают app restart без потери transcript.
- Parent видит child state/result без чтения чужих файлов.
- Человек видит orchestrator и child sessions в одном UI.

### STOP / fallback

Если Nimbalyst не проходит durability или APIs недоступны без форка:

1. проверить ACP client path;
2. оценить минимальный maintained fork;
3. только затем вернуться к собственной UI/runtime разработке.

---

## Milestone V2 — Agentplane external-agent validation

### Goal

Доказать, что Agentplane может быть deterministic workflow/evidence layer, не становясь вторым session host.

### V2.1 Local task

- [ ] Инициализировать Agentplane в test repository.
- [ ] Создать task, связанный с test GitHub issue.
- [ ] Получить PLANNER WorkOrder через external route.
- [ ] Передать WorkOrder Nimbalyst child session.
- [ ] Вернуть typed semantic result.
- [ ] Выполнить plan approval.

### V2.2 Implementation and evaluation

- [ ] Получить EXECUTOR episode.
- [ ] Выполнить в Nimbalyst worktree.
- [ ] Вернуть result.
- [ ] Получить EVALUATOR episode на другом provider.
- [ ] Проверить rework loop.
- [ ] Получить terminal state и ACR.

### V2.3 Failure paths

- [ ] stale fingerprint result rejected;
- [ ] child crashes before result;
- [ ] result malformed;
- [ ] observed check contradicts claimed check;
- [ ] effect-in-doubt;
- [ ] plan change invalidates approval.

### Evidence

```text
evidence/v2-agentplane/
├── task-mapping.md
├── workorders/
├── results/
├── receipts/
├── acr.json
└── failure-matrix.md
```

### Acceptance

Один Agentplane task полностью проходит через Nimbalyst sessions без ручного редактирования task state и без запуска Agentplane как конкурирующего global scheduler.

### STOP / fallback

Если external-agent loop требует слишком много ручной shell-хореографии:

- не интегрировать Agentplane managed runner;
- взять public schemas/patterns;
- реализовать минимальный compatible WorkOrder/Result/Receipt subset.

---

## Milestone V3 — GitHub Project scheduler spike

### Goal

Доказать динамическую issue-first логику до написания общего daemon.

### V3.1 Project fields

Создать/согласовать поля:

- [ ] Milestone;
- [ ] Priority;
- [ ] Phase;
- [ ] Status;
- [ ] Iteration;
- [ ] Team;
- [ ] Agent Profile;
- [ ] Automation;
- [ ] Agent State;
- [ ] Run ID;
- [ ] Risk;
- [ ] Human Gate.

### V3.2 Eligibility command

Прототип `acp issues eligible`:

- [ ] GraphQL read;
- [ ] existing board-admission checks;
- [ ] blockers/dependencies;
- [ ] normalized team/profile;
- [ ] deterministic sorting;
- [ ] explainable rejection reasons.

### V3.3 Claim/read-back

Прототип `acp issues claim <number>`:

- [ ] local SQLite transaction;
- [ ] unique active claim;
- [ ] fencing token;
- [ ] GitHub `Agent State/Run ID` projection;
- [ ] mandatory read-back;
- [ ] idempotent repeat;
- [ ] release/reconcile.

### V3.4 Changed issue reconciliation

- [ ] issue moved to terminal while run active;
- [ ] priority/team changed before claim;
- [ ] blocker reopened;
- [ ] Project API unavailable;
- [ ] GitHub update succeeded but acknowledgement was lost.

### Acceptance

Два concurrent claim attempts на один issue дают одного winner; повтор команды не создаёт второй run; GitHub и local state сходятся после reconnect.

---

## Milestone M1 — Thin control kernel

Starts only after V1–V3 are accepted.

### Goal

Объединить проверенные spikes в маленький local daemon/CLI.

### Packages

```text
packages/domain
packages/storage-sqlite
packages/github-adapter
packages/scheduler
packages/claim-manager
packages/agentplane-bridge
packages/control-mcp
packages/archive
apps/acpd
apps/acp-cli
```

### Commands

```text
acp doctor
acp issues list
acp issues eligible
acp issues claim
acp issues release
acp run status
acp run reconcile
acp run closeout
acp archive inspect
```

### Required behavior

- [ ] one writer daemon;
- [ ] SQLite migrations;
- [ ] append-only runtime events;
- [ ] transactional outbox;
- [ ] startup reconciliation;
- [ ] structured JSON output for all commands;
- [ ] no raw transcript duplication;
- [ ] no direct GitHub mutation outside outbox worker.

### Acceptance

Kill daemon after every state transition and restart. State either continues or becomes explicit `needs_recovery`; no duplicate execution appears.

---

## Milestone M2 — New Agent Team Skill

### Goal

Start using the flow through a visible Nimbalyst orchestrator before custom UI work.

### New skill responsibilities

- team/profile templates;
- planning and file-zone rules;
- GitHub issue context interpretation;
- Agentplane WorkOrder execution;
- structured decisions;
- child-session creation through `nimbalyst-host`;
- control MCP calls;
- human communication.

### Removed responsibilities

- inbox/outbox creation;
- watcher;
- headless runner ownership;
- NNN allocation;
- registry liveness;
- `delivery.log`;
- direct process spawning;
- direct issue claim;
- manual checkpoint requirement.

### Flow

```text
visible orchestrator session
  → control.next_assignment
  → Agentplane next step
  → spawn visible child session
  → observe result
  → resume Agentplane
  → checkpoint/closeout generated by host
```

### Acceptance

One real GitHub issue completes with planner, implementer and reviewer sessions; user can inspect every session and the run survives a Nimbalyst restart.

---

## Milestone M3 — Automatic archive and closeout

### Goal

Remove all dependence on agent memory for preserving work.

### Observer inputs

- Nimbalyst session status/result;
- raw transcript export or stable reference;
- edited files;
- Git status/head/diff;
- Agentplane task/evidence;
- GitHub issue/project snapshot;
- control runtime events.

### Automatic triggers

- [ ] session terminal;
- [ ] session waiting for human;
- [ ] Agentplane step transition;
- [ ] provider error;
- [ ] app/daemon shutdown;
- [ ] stale active session;
- [ ] run terminal;
- [ ] explicit archive.

### Outputs

Host-local full bundle:

```text
manifest.json
timeline.jsonl
session-links.json
transcript/*.jsonl
artifacts.json
decisions.jsonl
questions.jsonl
git-state.json
checks.json
closeout.md
```

Sanitized repository projection:

```text
.agent-control-plane/evidence/<issue>/<run>/README.md
.agent-control-plane/evidence/<issue>/<run>/acr.json
.agent-control-plane/evidence/<issue>/<run>/checks.json
```

### Acceptance

A child is forcibly terminated without calling any final tool. Observer still creates a complete bundle with explicit failure/unfinished disposition and preserves all transcript available to the host.

---

## Milestone M4 — Nimbalyst Issue Control panel

### Goal

Свести GitHub work board и Nimbalyst session board в один operator surface без форка приложения.

### MVP panel

- dispatchable queue;
- filters by Team/Priority/Status;
- claim/start/release;
- current run and Agentplane step;
- linked sessions and providers;
- last activity;
- pending human question;
- artifacts/checks;
- retry/cancel/answer/open-session controls.

### Constraints

- extension reads daemon API, not Nimbalyst internal DB schema;
- no second task database;
- every mutation calls typed daemon command;
- full read-back after mutation;
- panel loss does not stop daemon or sessions.

### Acceptance

Closing/reloading panel does not affect run. State is rebuilt from daemon/Nimbalyst links and no hidden UI-only status exists.

---

## Milestone M5 — Dynamic specialized teams

### Goal

Команды автоматически берут подходящие issues без превращения ролей в permanent processes.

### Policies

- profile eligibility from `Team`, labels and required capabilities;
- per-team concurrency;
- reviewer different from implementer;
- elevated/critical work requires plan approval;
- shared file-zone conflicts serialized;
- dependency-aware dispatch;
- provider/model cost tier;
- explicit manual override.

### First templates

```text
standard-feature
backend-change
frontend-change
research-and-decision
security-sensitive
release-operations
incident-investigation
```

### Acceptance

Queue with mixed backend/frontend/security issues dispatches only eligible work, preserves priority and never assigns two writers to overlapping exclusive zones.

---

## Milestone M6 — Legacy migration

### Goal

Выключить старый transport без потери accumulated history.

### Import

- [ ] inventory every legacy team and inbox;
- [ ] map messages to source/target/issue where possible;
- [ ] classify:
  - actionable;
  - done elsewhere;
  - superseded;
  - duplicate;
  - blocked external;
  - needs owner;
  - malformed;
  - orphaned;
  - archive only;
- [ ] create follow-up issues only for actionable items;
- [ ] freeze legacy archive read-only.

### Retirement

- [ ] disable watcher scripts;
- [ ] disable inbox runners;
- [ ] stop writing `registry.autopilot`;
- [ ] stop new MD transport;
- [ ] redirect typed sender to Control MCP/API;
- [ ] preserve live audit as migration/doctor tool.

### Acceptance

For one full operating cycle, no new file appears in legacy inbox/outbox and all new work is traceable from GitHub issue to run, sessions, worktree and closeout.

---

## Deferred milestones

Not part of initial product:

- remote A2A workers;
- multi-host supervisor;
- Postgres/Redis;
- Buzz integration;
- shared multi-user control plane;
- autonomous merge/deploy;
- provider marketplace;
- custom mobile client.

Trigger for reconsidering distributed infrastructure:

```text
two supervisors must concurrently claim work
or
executions run on more than one host
or
multiple operators need shared live control
```

Until then SQLite/local-first remains the selected architecture.

---

## First usable operating procedure

The earliest useful version does **not** wait for M4 panel:

1. GitHub Project is open in browser.
2. Nimbalyst is the session/operator window.
3. One visible `ACP Orchestrator` session is running per repository.
4. `acpd` selects and claims the next issue.
5. Orchestrator spawns child sessions and links them to the run.
6. Agentplane advances workflow and records evidence.
7. Nimbalyst saves transcripts automatically.
8. Archive coordinator creates closeout automatically.

This is the target we validate before building additional UX.
