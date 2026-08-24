# Agent Control Plane

Локальный control plane для управляемой работы команд coding-агентов поверх GitHub Issues, Nimbalyst и Agentplane.

## Решение

Проект **не строит новый desktop-клиент, новый чат и новый универсальный agent runtime с нуля**.

Он соединяет четыре уже существующих слоя:

1. **GitHub Issues + Projects** — каноническая очередь работы: приоритет, направление, зависимости, human gates и итоговый статус.
2. **Nimbalyst** — operator host: видимые Claude/Codex-сессии, workstreams, worktrees, transcript, tool calls, subagents, уведомления и ручное управление.
3. **Agentplane** — детерминированный lifecycle и evidence-контракт одной задачи: WorkOrder, state fingerprint, verification receipts, findings, rollback и Agent Change Record.
4. **Agent Control Plane** — тонкий локальный bridge/supervisor: выбор issue, claim/lease, привязка issue ↔ Agentplane task ↔ Nimbalyst sessions, outbox/reconciliation и автоматический closeout.

```text
GitHub Issues / Projects
        │
        ▼
Agent Control Plane
(issue scheduler + SQLite runtime registry + MCP)
        │
        ├────────► Agentplane
        │          task lifecycle + evidence
        │
        └────────► Nimbalyst
                   sessions + transcripts + UI
                         │
                         ├── Claude Code
                         ├── Codex
                         └── другие providers
```

## Главный принцип

У каждого вида состояния ровно один владелец:

| Состояние | Владелец |
|---|---|
| Что нужно сделать, приоритет, команда, бизнес-статус | GitHub Issue / Project |
| Какая локальная попытка сейчас владеет задачей | Agent Control Plane SQLite |
| План, bounded episodes, verification и evidence | Agentplane task |
| Разговор, tool calls, subagents и provider session | Nimbalyst |
| Код, ветка, diff и commits | Git / worktree |
| Человекочитаемый итог | generated closeout / ACR |

Markdown-файлы больше не используются как очередь сообщений или liveness-регистр.

## MVP

- один компьютер;
- один GitHub Project;
- Claude Code и Codex через Nimbalyst;
- максимум три одновременно активных issue-workstream;
- planner, implementer и independent reviewer как роли, а не постоянно живущие процессы;
- автоматическое сохранение transcript и действий в Nimbalyst;
- автоматический run manifest и closeout без требования, чтобы агент «не забыл чекпоинт»;
- raw transcripts не коммитятся в Git; в репозиторий попадает только очищенный evidence-пакет.

## Документы

- [`docs/research/2026-08-24-previous-audit.md`](docs/research/2026-08-24-previous-audit.md) — зафиксированный результат предыдущего внешнего аудита.
- [`docs/research/2026-08-24-ecosystem-and-skill-audit.md`](docs/research/2026-08-24-ecosystem-and-skill-audit.md) — аудит текущего Agent Team Skill и внешних решений.
- [`docs/architecture/0001-platform-strategy.md`](docs/architecture/0001-platform-strategy.md) — принятое архитектурное решение.
- [`docs/roadmap/ROADMAP.md`](docs/roadmap/ROADMAP.md) — последовательность проверочных спайков и реализации.

## Статус

`Architecture selected · implementation not started`
