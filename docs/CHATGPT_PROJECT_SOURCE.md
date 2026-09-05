# Расширенный контекст проекта ChatGPT Web — MsgHub

## Назначение документа

Этот файл предназначен для добавления в Sources проекта ChatGPT Web вместе с `github_project_bootstrap.md`.

Короткие Project Instructions должны содержать только базовые неизменяемые правила и ссылаться на этот документ. Сам этот файл хранит расширенный рабочий контекст, который не нужно помещать в ограниченное поле инструкций.

Важно: для динамических фактов этот документ не является источником истины. Актуальные branch/HEAD, Issues, PR, код, CI, версии и статус реализации всегда перечитываются из GitHub.

## Проект

Целевой репозиторий: `dilukhin/msghub`.

MsgHub — самохостинговая система ретрансляции сообщений между чатами разных платформ. Первая цель — надёжный двусторонний мост Telegram ↔ WhatsApp для обычных групп на небольшой Ubuntu VPS. В дальнейшем планируются VK, MAX и другие адаптеры.

## Язык

Русский — основной язык проекта и общения.

На русском должны быть:

- ответы пользователю и оператору;
- README и документация;
- Issues, описания PR и commit messages;
- CLI/`--help`;
- логи, диагностика, предупреждения и тексты ошибок;
- комментарии и docstring собственного кода.

Идентификаторы в коде, машинные поля и названия внешних API могут оставаться английскими, если этого требует совместимость или однозначность. Протокольные имена не переводить ценой несовместимости.

## Начало задачи

Для GitHub-задачи:

1. Прочитать `github_project_bootstrap.md` из Sources.
2. Через GitHub Connector загрузить `dist/projects/dilukhin__msghub.md` из `dilukhin/github-connector-knowledge`.
3. Проверить, что runtime bundle относится к `dilukhin/msghub`.
4. Перечитать актуальный `main`, связанный Issue/PR и затронутые файлы.
5. При необходимости изучить `AGENTS.md`, `README.md`, `docs/ARCHITECTURE.md`, `docs/MVP_V0.1.md`, `docs/ROADMAP.md`, `docs/DEVELOPMENT.md`.

Старые SHA, handoff, память диалога и прежние эксперименты могут быть навигацией, но не заменяют GitHub read-back.

## GitHub workflow

GitHub Connector — первичный и штатный remote-транспорт ChatGPT Web.

Не выполнять пробные `git`/`gh` remote-команды. Не считать наличие shell или локального Git доказательством удалённого доступа.

Обычная разработка:

```text
актуальный main
→ ветка agent/...
→ логические commits
→ PR
→ review/checks
→ merge
→ GitHub-side read-back
```

`main` напрямую не изменять при обычной разработке.

Для многофайловой записи через Connector использовать:

```text
blob → tree → commit → ref
```

Перед `update_ref` перечитать HEAD. После значимой write-операции выполнить один целевой read-back.

Если необходимая Connector-операция отсутствует или подтверждённо заблокирована, не менять transport молча. Применить fallback из `github_project_bootstrap.md`.

## Архитектурные инварианты

1. Relay core остаётся платформонезависимым.
2. Telegram-, WhatsApp-, VK- и MAX-specific объекты не становятся моделью ядра, кроме ограниченных adapter metadata.
3. Не создавать попарные маршрутизаторы `Telegram → WhatsApp`, `Telegram → MAX` и т. п. Endpoint-ы объединяются через `LogicalRoom` и каноническую модель событий.
4. Accepted ingress и необходимые `Delivery` records должны быть долговечно зафиксированы до признания события принятым.
5. Loop prevention, deduplication и `MessageLink` должны переживать restart процесса и VPS.
6. `uncertain` означает неизвестность факта remote acceptance. Его нельзя автоматически превращать в обычный retry без reconciliation или отдельно принятой политики.
7. SQLite/WAL — исходное долговременное хранилище v0.1.
8. Media cache ограничен и является временным состоянием доставки, а не постоянным архивом.
9. WhatsApp Web-compatible transport считать заменяемой зависимостью повышенного риска и изолировать от relay core отдельным процессом/зоной отказа.
10. Целевая среда v0.1 — небольшая Ubuntu VPS порядка 1–2 vCPU, 1–2 GiB RAM и около 20 GiB SSD.

Без измеренной необходимости не делать обязательными Redis, RabbitMQ, Kafka, PostgreSQL, S3, Docker/Kubernetes и другие дополнительные службы.

## Приватность и секреты

MsgHub обрабатывает частные сообщения, поэтому обычная диагностика должна по возможности работать с идентификаторами и метаданными, а не с содержимым сообщений.

Не публиковать в Git, Issues, PR и обычные логи:

- tokens;
- cookies;
- private keys;
- pairing/session material;
- QR/pairing credentials;
- runtime databases;
- media cache;
- полный текст приватных сообщений;
- содержимое вложений.

Диагностика по умолчанию должна предпочитать:

- event/message IDs;
- platform/endpoint;
- timestamps;
- размеры payload;
- MIME/type;
- delivery states;
- error classes/codes;
- retry/reconciliation state.

Полный private payload допустим только в специально включённом диагностическом режиме с явным предупреждением и без credentials.

## Надёжность

Happy path не считается достаточным.

Для изменений, влияющих на доставку, отдельно рассматривать и по возможности тестировать:

- duplicate ingress;
- restart процесса/VPS;
- network loss;
- rate limit;
- retryable failure;
- terminal failure;
- ambiguous send / `uncertain`;
- adapter reconnect;
- SQLite reopen/crash recovery;
- media quota exhaustion.

Отказ одного адаптера не должен повреждать состояние остальных endpoint-ов и core database.

## Scope и Issues

Перед работой прочитать Issue полностью и соблюдать его зависимости.

Не расширять scope без отдельного решения. Если новая работа существенно независима или меняет архитектуру, оформить отдельный Issue вместо скрытого расширения текущего PR.

Текущий начальный граф:

```text
#2 core contract ─────────────┐
                             ├→ #5 core ───────┐
#3 WhatsApp feasibility ─────┼→ #7 WhatsApp ──┤
                             │                 ├→ #8 text E2E → #9 media → #10 v0.1
#4 Telegram contract ────────┴→ #6 Telegram ──┘
```

Issues #2, #3 и #4 могут идти параллельно.

До завершения #2 не фиксировать язык реализации, конкретный IPC и библиотеку WhatsApp как необратимые решения.

После выбора стека обновить конкретные test commands и CI requirements в:

- `AGENTS.md`;
- `docs/DEVELOPMENT.md`;
- `dilukhin/github-connector-knowledge/src/profiles/dilukhin__msghub.yaml`.

## Документация решений

Изменения следующих контрактов требуют синхронного обновления документации и тестов:

- `CanonicalEvent`;
- `LogicalRoom` / `Endpoint`;
- `Delivery` state machine;
- `MessageLink` / reply mapping;
- adapter capabilities;
- SQLite schema/migrations;
- media lifecycle/quota;
- IPC core ↔ isolated adapter;
- security/privacy boundary;
- operator-visible diagnostics.

Существенные решения с альтернативами и компромиссами фиксировать в ADR после утверждения формата в Issue #2.

## Завершение задачи

В финале кратко указать:

- что изменено;
- какие проверки выполнены;
- состояние PR/CI;
- фактический GitHub-side read-back;
- появились ли новые переносимые замечания по GitHub Connector/API.

Если новых таких наблюдений нет, написать:

«Новых неучтённых замечаний по работе с GitHub не выявлено.»
