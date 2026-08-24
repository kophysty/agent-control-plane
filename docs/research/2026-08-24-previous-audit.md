# Предыдущий внешний аудит Agent Team

**Дата фиксации:** 2026-08-24  
**Источник:** внешний аудит `kophysty/skills` checkpoint `ca1763d` и подтверждающие инциденты из проектов, использовавших Agent Team.  
**Назначение документа:** сохранить исходный диагноз до нового исследования готовых решений и выбора архитектуры.

## Executive verdict

Текущий Agent Team:

- **NO-GO** как надёжная автономная система управления командами агентов;
- **GO** как экспериментальный протокол, audit trail и набор ценных workflow-контрактов;
- по масштабу уже является не skill, а самописным agent runtime/control plane.

Изначальная идея — человекочитаемые сообщения в Markdown — была рабочей для ручного handoff. Проблема появилась, когда поверх неё начали строиться:

- очередь работ;
- delivery и ACK;
- liveness;
- watcher/runner;
- retries;
- lifecycle команд;
- Git/worktree coordination;
- GitHub Actions;
- admission gates;
- operator widget;
- поддержка Claude и Codex.

Файлы остались одновременно транспортом, журналом, runtime state и audit artifact. Из-за этого каждое новое защитное поле создавало новые варианты рассинхронизации.

## Основной архитектурный дефект

> У execution нет единственного транзакционного владельца.

Одна операция отражается сразу в нескольких местах:

- message frontmatter;
- наличие файла в inbox;
- `status: unread/read/done`;
- `delivery.log`;
- `journal.md`;
- `registry.json`;
- lock/PID;
- Git commit и branch;
- GitHub issue/project;
- интерактивный чат.

Обновить их атомарно невозможно. Поэтому наблюдались состояния вида:

```text
journal говорит «отправлено», delivery.log ничего не знает;
message уже unread, но содержимое ещё меняется;
registry говорит autopilot=true, process умер;
process жив, registry говорит false;
watcher увидел сообщение, но session не проснулась;
runner работает, но человек видит полную тишину.
```

## Watcher и runner

### Watcher

Watcher умел обнаружить изменение файла. Он **не умел продолжить уже открытую Codex-сессию**. Завершение фонового PowerShell-процесса не является событием для интерактивного чата.

Возврат к watcher-only поэтому не возвращает прежнюю видимость. Он возвращает только file detection.

### Runner

Headless runner умеет:

- найти unread;
- запустить отдельный bounded turn;
- получить ответ;
- переложить файл;
- изменить status;
- продолжить цикл.

Но это другой процесс и другая session. Работа может происходить, а пользовательский чат остаётся неподвижным.

Нельзя называть это interactive wakeup. Нужен отдельный operator UI или host, который владеет session lifecycle и stream событий.

## Подтверждённые системные симптомы

### Неправдивая unread-очередь

Большая часть накопившихся unread находилась у активных команд, а не у архивных. При этом `status` изменялся вручную, поэтому число не позволяло отличить:

- реально непрочитанное;
- фактически обработанное без правки frontmatter;
- superseded;
- выполненное в другом issue/PR;
- orphaned.

### Самоколлизия runtime

В одном инциденте одновременно существовали:

- несколько runner одной роли;
- интерактивный и headless executor одной роли;
- шесть watcher одного inbox;
- десятки процессов Claude/PowerShell;
- два исполнителя, пишущие в один worktree.

Runtime не имел единственного supervisor и мог самовосстанавливаться после попытки остановки дочернего процесса.

### Расхождение транспортной правды

Один task одновременно был:

- `draft`, затем `unread` в файле;
- `sent` по journal;
- несуществующим по `delivery.log`.

Это доказывает, что три носителя нельзя считать одной state machine.

### Git как runtime и архив

Git полезен для review и долговечных решений, но не для mutable liveness. Дополнительно содержимое agent-team сообщений уже переносило секреты в долговечную Git-историю. Поэтому raw transcript и runtime events не должны автоматически коммититься.

### Self-hosting blindness

Skill проверялся механизмами, которые сам менял:

- manifest закрытия создал цикл;
- checklist не проверил собственное требование;
- аудит выполнялся subagents и обходил заявленный inbox-контракт;
- внутренний аудитор находился в том же failure domain.

Self-hosting полезен как stress-test, но не как единственное доказательство корректности.

## Что в текущем skill ценно

Нужно сохранить и перенести в новую систему:

- явное разделение roadmap → plan → feature → slice;
- dependencies и file zones до параллельной выдачи;
- human gates;
- независимый reviewer/evaluator;
- typed handoff;
- `delivery_id` / idempotency;
- at-least-once вместо ложного exactly-once;
- explicit `needs_human`;
- fail-closed admission;
- live reconciliation;
- sanitized evidence;
- closure dispositions;
- GitHub identity isolation;
- read-only operator projection.

## Что нельзя переносить как runtime

- inbox/outbox как очередь;
- ручной `status: unread`;
- NNN как identity;
- `registry.json` как liveness truth;
- `delivery.log` как отдельную state machine;
- watcher как wakeup;
- агента как владельца runner;
- Git commit как ACK;
- GitHub Actions как локальный process monitor;
- требование агенту самому не забыть checkpoint.

## Исходная целевая модель аудита

Предыдущий аудит предложил:

```text
GitHub Issues / Project  = истина о работе
Supervisor               = истина об исполнении
SQLite                    = structured runtime state
Provider adapters         = Claude/Codex sessions
Operator UI               = видимая timeline
Git / Markdown            = sanitized audit projection
```

Новый research уточнил, что значительную часть этой модели не нужно писать с нуля: Nimbalyst, Agentplane и Symphony уже закрывают разные слои. Итоговая сборка описана в следующих документах.

## Acceptance boundary

Новая система не считается рабочей, пока black-box тесты не подтверждают:

1. crash после claim не теряет задачу;
2. повторный command не создаёт вторую execution;
3. одна задача и один worktree имеют одного активного владельца;
4. старый process после потери lease не может записать terminal result;
5. UI/restart не прерывает agent execution и не теряет transcript;
6. любой вопрос человеку виден вне provider terminal;
7. terminal session автоматически получает closeout bundle;
8. secret redaction происходит до Git/GitHub projection;
9. watcher/event loss компенсируется reconciliation;
10. active execution всегда видна человеку с issue, provider, session, workspace и last activity.
