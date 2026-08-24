# Ecosystem и code-аудит Agent Team

**Дата:** 2026-08-24  
**Объекты:** `kophysty/skills/Skill/agent-team`, checkpoint lifecycle-v9, Agentplane, Nimbalyst, OpenAI Symphony, AutoForge, Automaker, ACP adapters, Buzz и соседние coding-agent managers.

## 1. Краткий вывод

Готового единственного продукта, который одновременно идеально решает:

- GitHub issue scheduling;
- Claude/Codex sessions;
- multi-agent team decomposition;
- persistent transcripts;
- worktrees;
- review/evidence;
- human approvals;
- operator UI;
- automatic closeout;

не найдено.

Но писать всю систему с нуля также не нужно. Рынок уже разделил задачу на три зрелых класса:

1. **Session host / operator UI** — лучше всего подходит Nimbalyst.
2. **Deterministic task lifecycle / evidence** — лучше всего подходит Agentplane.
3. **Issue-driven scheduler / supervisor semantics** — лучший открытый blueprint даёт OpenAI Symphony.

Наш проект должен быть тонким соединительным слоем между ними.

## 2. Аудит текущего `agent-team`

### 2.1. Фактический масштаб

Текущий каталог — это уже не небольшой skill:

- `SKILL.md` — около 125 КБ;
- `references/protocol.md` — около 155 КБ;
- крупный набор PowerShell/bash runtime scripts;
- live audit, board admission, protocol pilot, GitHub identity, safe materialization;
- десятки validation suites;
- отдельный Tauri/React widget и read-model contract.

Это полноценный экспериментальный control plane, размещённый в форме skill.

### 2.2. Сильные части

#### Planning contract

`references/planning-protocol.md` правильно разделяет:

```text
roadmap → plan → features → slices
```

Также правильно зафиксированы два условия параллельности:

1. непересекающиеся file zones;
2. закрытые dependencies.

Эта модель переносится в новую систему практически без изменений.

#### GitHub admission

`check-board-admission.ps1` уже реализует fail-closed read-only проверку GitHub Project:

- Milestone;
- Priority;
- Phase;
- Status;
- Iteration для активной работы;
- Size/Estimate как warnings;
- явный `NOT_EVALUATED`, когда нет credential/network/schema truth.

Это хороший фундамент будущего issue eligibility policy.

#### Typed transport ideas

`send-agent-team-message.ps1` содержит полезные решения:

- typed source/target;
- explicit roles;
- `delivery_id`;
- v5/v6 compatibility;
- canonical root resolver;
- duplicate detection;
- temp file + atomic rename;
- deadline/escalation semantics.

Проблема не в этих контрактах, а в том, что они реализованы поверх нескольких файловых state stores.

#### Honest live audit

`read-model-contract.md` правильно запрещает выводить `LIVE_CONFIRMED` по одному PID и отличает:

- physical state;
- declared state;
- index state;
- consumer confidence.

Сам widget также разумно ограничен read-only режимом.

### 2.3. Непереносимые runtime-компоненты

#### `send-agent-team-message.ps1`

Скрипт вынужден:

- сканировать Markdown inbox для поиска `delivery_id`;
- брать process-wide mutex для NNN;
- отдельно писать envelope;
- отдельно писать runtime JSON;
- отдельно отражать состояние в logs.

То есть доставка не является одной транзакцией.

#### `runtime-state.ps1`

Фактически реализует файловую БД:

- leases;
- claims;
- dispatch records;
- completion;
- escalation;
- archive gates;
- heartbeats;
- PID/proc-start verification.

Каждая сущность находится в отдельном JSON/каталоге. Это полезная спецификация state machine, но не оптимальная persistence model.

#### `codex-inbox-runner.ps1`

Один скрипт одновременно выполняет роли:

- queue consumer;
- process supervisor;
- timeout/kill manager;
- retry engine;
- protocol interpreter;
- response router;
- sequence allocator;
- message state mutator;
- logger;
- journal writer;
- registry writer.

Это главный источник нелокальных ошибок: исправление одной обязанности меняет поведение остальных.

#### `run-codex-turn.ps1` и `run-claude-turn.ps1`

Каждое сообщение запускает отдельный headless turn (`codex exec`, `claude -p`). Скрипты не владеют общей provider session и не получают единую долговечную event stream. Для Windows Codex используется `danger-full-access`, а ограничения фактически держатся на prompt-контракте.

Это нельзя использовать как долгосрочный multi-provider runtime.

### 2.4. GitHub Issues: что уже есть и чего нет

В skill записано `workflow: issues`; планы и roadmap считаются проектной истиной, а `.agent-team` — транспортом.

Реализовано:

- проверка issue/project fields;
- rules о roadmap/feature/slice;
- ссылки между task и tracker.

Не реализовано как единый механизм:

- получение упорядоченной dispatchable очереди;
- атомарный claim;
- lease;
- reconciliation с изменившимся issue;
- остановка execution, если issue стал неактивным;
- retry/backoff;
- автоматический выбор специализированного профиля;
- глобальные concurrency limits.

Это именно тот слой, который формализует Symphony.

## 3. Внешние решения

## 3.1. Nimbalyst

Repository: <https://github.com/nimbalyst/nimbalyst>

### Что уже закрывает

- desktop UI для Claude Code, Codex, OpenCode и Copilot;
- параллельные sessions;
- session Kanban;
- search/resume/archive;
- workstreams и worktrees;
- edited files и visual diff review;
- mobile dashboard и ответы на blocking prompts;
- OS/push notifications;
- extension panels и AI tools.

### Самое важное для нашей задачи

Nimbalyst сохраняет provider-native raw messages в append-only `ai_agent_messages`. Это persisted source of truth для transcript. Canonical UI events восстанавливаются из raw log через provider-specific parsers.

Поддерживаются отдельные parsers для:

- Claude Code;
- Codex SDK/App Server;
- Codex ACP;
- Copilot;
- OpenCode.

Это непосредственно закрывает боль:

> агент или subagent завершился, но transcript, tool calls и результат не были сохранены.

Raw события сохраняет host, а не модель по собственной дисциплине.

### Cross-session orchestration

Встроенный `nimbalyst-host` MCP уже содержит:

- `create_session`;
- `spawn_session`;
- `send_prompt`;
- `get_session_status`;
- `get_session_result`;
- `list_queued_prompts`;
- `respond_to_prompt`;
- `list_spawned_sessions`;
- `list_worktrees`;
- `notify_user`;
- cross-session summaries и workstream overview;
- scheduled wakeups.

Следовательно, существующий Agent Team можно перенести из file inbox в видимые Nimbalyst sessions без разработки нового chat/runtime UI.

### Ограничения

- tracker и collaboration имеют собственную evolving sync model;
- часть meta-agent surfaces является внутренним host MCP, а не обещанным стабильным внешним API;
- extension DB access read-only и schema не считается стабильной;
- исторически были cross-project tracker/sync дефекты;
- Nimbalyst не является строгим GitHub issue scheduler и evidence supervisor.

### Решение

**Выбрать Nimbalyst как operator host и transcript archive. Не форкать на первом этапе.**

Интеграцию делать через:

1. обновлённый Agent Team Skill, который использует `nimbalyst-host` MCP;
2. наш execution-scoped MCP;
3. затем небольшой Nimbalyst extension panel.

## 3.2. Agentplane

Repository: <https://github.com/basilisk-labs/agentplane>

### Сильная сторона

Agentplane очень точно проводит нужную границу:

| Actor | Responsibility |
|---|---|
| Human | outcome, material risk, approval |
| Agent | semantic reasoning, design, implementation, evaluation |
| CLI | lifecycle, authority, Git/PR routing, evidence, recovery, closure |

Его ключевые контракты:

- `StateFingerprint`;
- `AgentWorkOrder`;
- `AgentSemanticResult`;
- `ExecutionReceipt`;
- `WorkflowStep`;
- Agent Change Record.

Особенно ценно разделение:

```text
что агент сообщил
≠
что supervisor самостоятельно наблюдал
```

Agent-reported tests не становятся verified evidence без observer receipt.

### Что он решает

- plan approval;
- bounded semantic episodes;
- stale result rejection;
- direct/branch_pr workflows;
- worktree and PR mechanics;
- verification/evaluator/rework;
- effect-in-doubt;
- rollback/findings;
- durable task artifact and ACR.

### Что он не решает целиком

- богатый session UI;
- universal provider transcript archive;
- GitHub Project priority polling в OSS happy path;
- постоянно работающую команду специализированных agent profiles;
- multi-session human-visible room.

Task state остаётся repo-local Markdown/frontmatter, но механически изменяется CLI и имеет revisions/fingerprints — это принципиально надёжнее ручного inbox protocol.

### Решение

**Не делать Agentplane вторым глобальным scheduler.**

Использовать его как:

- lifecycle/evidence engine одной GitHub issue;
- источник public schemas и проверенных паттернов;
- optional CLI dependency после integration spike.

GitHub остаётся источником work priority/status, Nimbalyst — session truth, Agentplane — per-task workflow/evidence.

## 3.3. OpenAI Symphony

Repository: <https://github.com/openai/symphony>

Symphony — наиболее точный blueprint для динамической очереди:

- continuously reads issue tracker;
- normalized Issue model;
- `dispatchable` eligibility;
- priority ordering;
- blockers/labels/assignee/state;
- bounded concurrency;
- per-issue workspace;
- one authoritative orchestrator state;
- retry/backoff;
- reconciliation;
- stop when issue becomes ineligible;
- repository-owned `WORKFLOW.md`.

Важно: сама спецификация сознательно не делает rich UI и не требует persistent DB. Поэтому Symphony не заменяет Nimbalyst и не закрывает наш automatic archive.

### Решение

**Использовать Symphony как scheduler contract, а не строить вокруг reference implementation весь продукт.**

Наш GitHub adapter повторяет его normalized issue/dispatch/reconcile semantics, но хранит structured local runtime в SQLite и запускает Nimbalyst workstreams.

## 3.4. AutoForge

Repository: <https://github.com/AutoForgeAI/autoforge>

Предыдущий вывод «репозиторий неактивен, значит архитектура не работает» неверен.

AutoForge использует:

- SQLite/SQLAlchemy;
- atomic feature claim/update;
- dependency graph;
- deterministic parallel orchestrator;
- feature-management MCP;
- FastAPI/WebSocket UI;
- separate coding and regression agents.

README объясняет снижение необходимости проекта тем, что современные agent runtimes начали включать long-running harness, а не провалом SQLite/MCP модели.

### Урок

Правильная формула AutoForge сохраняется:

```text
orchestrator chooses work
agent receives assigned feature
database owns state
MCP exposes bounded actions
UI streams progress
```

Но готовый продукт уже уже по возможностям, чем Nimbalyst + Agentplane.

## 3.5. Automaker

Repository: <https://github.com/AutoMaker-Org/automaker>

Полезный архитектурный урок: agent service был вынесен в долговечный Electron main process, чтобы web renderer мог перезапускаться, а agent execution и session history продолжали жить.

Сильные части:

- multiple providers;
- workspace/worktree UI;
- real-time streaming;
- plan/review;
- persistent main-process runtime.

Слабые части внутреннего аудита проекта:

- file-per-feature state;
- distributed state transitions;
- недостаточно протестированные provider edges;
- concurrency/partial-write risks.

### Урок

Долговечный host process и reconnecting UI — правильно. File-per-feature operational state — не переносим.

## 3.6. Vibe Kanban

Repository: <https://github.com/BloopAI/vibe-kanban>

Проект имел очень близкий UX:

- kanban issues;
- workspace per task;
- 10+ agents;
- diff review;
- branch/PR;
- preview browser.

Но проект официально sunsetting. Это не означает, что UX был неверным; это означает, что зависеть от него как от стратегической основы сейчас рискованно.

Отдельный полезный урок проекта: большие append-only execution logs были вынесены из SQLite в JSONL, потому что они создавали locks и занимали почти весь DB size. Для нашей системы это подтверждает разделение:

```text
SQLite = structured state / indexes
raw provider stream = host archive / append-only bundle
```

Nimbalyst уже владеет raw provider archive, поэтому второй полный transcript store создавать не нужно.

## 3.7. Claude Squad

Repository: <https://github.com/smtg-ai/claude-squad>

Хорошо решает:

- tmux/TUI sessions;
- worktrees;
- parallel agents;
- review before apply.

Не решает в нужной полноте:

- issue scheduler;
- typed lifecycle;
- evidence/ACR;
- persistent cross-provider transcript archive;
- automatic closeout.

Использовать как UX/reference, не как основу.

## 3.8. Buzz и ACP

Repositories:

- <https://github.com/block/buzz>
- <https://github.com/agentclientprotocol/claude-agent-acp>
- <https://github.com/agentclientprotocol/codex-acp>

Buzz подтверждает правильное protocol separation:

```text
ACP = client ↔ agent runtime
MCP = agent ↔ tools
relay/event log = shared collaboration truth
```

Claude/Codex ACP adapters уже нормализуют provider sessions и tool events. Они полезны как будущий provider-neutral слой, но Nimbalyst уже поддерживает оба provider напрямую. Не нужно вставлять ACP ещё одним обязательным hop в MVP, если host-native integration даёт более полный transcript.

Buzz целиком избыточен для single-user local-first MVP: Postgres, Redis, signed events, communities и forge создадут гораздо более глубокую разработку.

## 3.9. Другие близкие проекты

### Contrabass

Symphony-inspired supervisor с GitHub Issues/Linear/internal board, worktrees, recovery, TUI/web и team phases. Полезен как reference реализации GitHub scheduler, но будет конкурировать с Nimbalyst за роль runtime/UI.

### oh-my-symphony

Multi-model Symphony fork с SQLite leases и UI/TUI. Молодой проект; полезен для изучения provider abstraction, не для основной зависимости.

### Kōbō

Очень близкая локальная архитектура: Claude Agent SDK, Codex App Server, SQLite WAL, WebSocket, MCP, session timeline и scheduled wakeups. Но проект новый и малораспространённый; использовать как reference, не как dependency.

## 4. Сравнительная матрица

| Solution | Issue scheduler | Session UI | Transcript | Multi-provider | Worktree | Evidence/gates | Решение |
|---|---:|---:|---:|---:|---:|---:|---|
| Current Agent Team | partial | weak widget | MD/log partial | Claude/Codex headless | yes | extensive rules | migrate contracts only |
| Nimbalyst | local tasks, not strict GitHub queue | **strong** | **strong raw archive** | **strong** | **strong** | medium | operator host |
| Agentplane | per-task backend/sync | CLI | run artifacts | runner-dependent | **strong** | **strongest** | lifecycle/evidence |
| Symphony | **strong blueprint** | logs/optional | limited | Codex-oriented spec | **strong** | workflow-defined | scheduler semantics |
| AutoForge | feature DB queue | web | logs/progress | mostly Claude/custom | limited | tests | architectural reference |
| Automaker | board/auto mode | strong | sessions | strong | strong | medium | reference only |
| Vibe Kanban | strong UX | strong | execution logs | strong | strong | review-centric | not strategic dependency |
| Claude Squad | manual | TUI | session terminal | several CLIs | strong | weak | reference only |
| Buzz | workflows/jobs | strong shared rooms | event log | ACP | agent-defined | approvals/audit | future remote collaboration |

## 5. Финальный выбор

### Не строим

- новый desktop UI;
- собственный chat protocol;
- новую provider session database;
- watcher/runner framework;
- ещё одну Markdown queue;
- multi-host distributed platform;
- универсальный A2A/ACP gateway в MVP.

### Используем

- **Nimbalyst** — sessions, transcripts, workstreams, worktrees, notification, operator UI;
- **GitHub Issues/Projects** — business work queue;
- **Symphony semantics** — eligibility, priority, claim, retry, reconciliation;
- **Agentplane** — WorkOrder/result/receipt/evidence/ACR;
- **наш SQLite daemon/MCP** — только runtime links, leases и outbox.

## 6. Риски, которые должны быть сняты спайками

1. Nimbalyst child session действительно сохраняет полный Claude/Codex transcript, tool calls и nested subagents после restart.
2. `nimbalyst-host` meta-agent tools доступны в обычном пользовательском workspace и не требуют форка.
3. Можно детерминированно связать GitHub issue, Nimbalyst workstream/session и worktree.
4. Agentplane external-agent loop можно выполнить из Nimbalyst session без ручной shell-хореографии.
5. Nimbalyst extension может безопасно показать локальное control-plane состояние, не читая нестабильные внутренние таблицы напрямую.
6. Session closeout можно создать observer-ом даже если child agent не вызвал финальный MCP tool.
7. Raw transcript можно экспортировать/архивировать с redaction до любой Git projection.

Пока эти семь спайков не пройдены, глубокая реализация supervisor запрещена.

## 7. Источники

### User repositories

- <https://github.com/kophysty/skills/tree/main/Skill/agent-team>
- <https://github.com/kophysty/skills/blob/main/.agent-team/agent-team-lifecycle-v9/CHECKPOINT-2026-08-24-external-audit.md>

### Primary external sources

- <https://github.com/nimbalyst/nimbalyst>
- <https://github.com/nimbalyst/nimbalyst/blob/main/docs/TRANSCRIPT_ARCHITECTURE.md>
- <https://github.com/nimbalyst/nimbalyst/blob/main/docs/THE_HARNESS.md>
- <https://github.com/basilisk-labs/agentplane>
- <https://github.com/basilisk-labs/agentplane/blob/main/docs/developer/architecture.mdx>
- <https://github.com/basilisk-labs/agentplane/blob/main/docs/user/task-lifecycle.mdx>
- <https://github.com/openai/symphony>
- <https://github.com/openai/symphony/blob/main/SPEC.md>
- <https://github.com/AutoForgeAI/autoforge>
- <https://github.com/AutoMaker-Org/automaker>
- <https://github.com/BloopAI/vibe-kanban>
- <https://github.com/smtg-ai/claude-squad>
- <https://github.com/block/buzz>
