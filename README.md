# Agent Control Plane

Исследование и минимальный интеграционный слой для управляемой работы команд coding-агентов поверх GitHub Issues, Nimbalyst и, при необходимости, Agentplane.

## Текущее решение

Проект **не начинает с разработки нового desktop-клиента, отдельного чата, глобального scheduler или универсального agent runtime**.

Ближайший рабочий маршрут:

1. **GitHub Issues + Projects** остаются канонической очередью работы: приоритет, зависимости, acceptance, PR/CI и бизнес-статус.
2. **Nimbalyst** становится operator cockpit и заменяет runtime-часть старого Agent Team Skill: видимые Claude/Codex-сессии, workstreams, worktrees, transcript, tool calls, subagents, вопросы, уведомления и ручное управление.
3. **Agent Team** сокращается до methodology/policy skill: planning, роли, file zones, dependencies, независимый review и human gates — без inbox/watcher/runner.
4. **Agentplane** пилотируется только для критических задач, где нужны bounded WorkOrders, stale-result rejection, independently observed verification и ACR.
5. Собственный **`acpd`/SQLite scheduler/bridge откладывается**, пока реальная эксплуатация не покажет повторяющуюся проблему claim, reconciliation, automatic closeout или unattended dispatch.

```text
GitHub Issues / Projects
        │
        ▼
Nimbalyst workstream
  ├── planner session
  ├── implementer session
  └── independent reviewer session
        │
        ├── existing Git / PR / CI
        └── Agentplane only for critical work
```

## Главный принцип

У каждого вида состояния один владелец:

| Состояние | Владелец на ближайшем этапе |
|---|---|
| Что нужно сделать, приоритет, acceptance и бизнес-статус | GitHub Issue / Project |
| Кто сейчас работает и что происходит в сессиях | Nimbalyst |
| Разговор, tool calls, subagents и provider session | Nimbalyst raw transcript |
| Код, branch, diff, commits и hosted checks | Git / GitHub |
| Формальный controlled lifecycle и evidence критической задачи | Agentplane, выборочно |
| Человекочитаемый итог | Issue/PR + sanitized closeout при необходимости |

Markdown-файлы больше не рассматриваются как очередь сообщений или liveness-регистр.

## Immediate pilot

- один компьютер и один GitHub Project;
- несколько реальных дней работы в Nimbalyst, а не только synthetic demo;
- Claude Code и Codex session durability после restart;
- parent/child orchestration и отдельные worktrees;
- tool calls, thinking/reasoning output, questions и edited files видимы;
- один стандартный issue проходит planner → implementer → reviewer;
- один критический issue отдельно проходит Agentplane spike;
- никакого нового daemon/UI до результатов этих двух сценариев.

## Документы

- [`docs/research/2026-08-24-previous-audit.md`](docs/research/2026-08-24-previous-audit.md) — исходный внешний аудит Agent Team.
- [`docs/research/2026-08-24-ecosystem-and-skill-audit.md`](docs/research/2026-08-24-ecosystem-and-skill-audit.md) — code-аудит skill и внешних решений.
- [`docs/research/2026-08-24-agent-control-plane-and-nimbalyst-landscape.md`](docs/research/2026-08-24-agent-control-plane-and-nimbalyst-landscape.md) — дополнительное исследование governance/workbench/runtime альтернатив.
- [`docs/architecture/0001-platform-strategy.md`](docs/architecture/0001-platform-strategy.md) — долгосрочная layered architecture.
- [`docs/architecture/0002-nimbalyst-first-operating-route.md`](docs/architecture/0002-nimbalyst-first-operating-route.md) — актуальный минимальный маршрут: Nimbalyst first, GitHub retained, Agentplane selective.
- [`docs/roadmap/ROADMAP.md`](docs/roadmap/ROADMAP.md) — исходный validation-first roadmap; V2/V3 теперь рассматриваются как условные этапы после Nimbalyst pilot.

## Статус

`Nimbalyst-first pilot selected · custom control-plane implementation deferred`