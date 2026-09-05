# Правила разработки MsgHub

## Назначение

Этот документ задаёт рабочие правила до появления полноценного кода. Архитектурные детали находятся в `ARCHITECTURE.md`, границы первой версии — в `MVP_V0.1.md`, порядок работ — в `ROADMAP.md`.

## Язык

Русский является основным языком проекта и всей человекочитаемой поверхности. Исключение — технические идентификаторы и внешние протокольные имена, которые нельзя или нецелесообразно переводить.

## Ветвление и публикация

- `main` — интеграционная ветка.
- Для задач использовать ветки `agent/<краткое-название>`.
- Изменения публиковать через PR.
- Один PR должен иметь понятный scope и ссылку на Issue, если работа ведётся по Issue.
- Commit messages писать по-русски и делать логическими, а не механически «по одному файлу».
- Force-update `main` запрещён.

## Документация решений

Изменение одного из следующих контрактов требует обновления документации до merge:

- CanonicalEvent;
- LogicalRoom / Endpoint;
- Delivery state machine;
- MessageLink / reply mapping;
- adapter capabilities;
- SQLite schema/migrations;
- media lifecycle/quota;
- IPC relay core <-> isolated adapter;
- security/privacy boundary;
- operator-visible diagnostics.

Существенные решения, для которых были реальные альтернативы и компромиссы, следует оформлять ADR в `docs/adr/` после утверждения формата в Issue #2.

## Надёжность

Happy path недостаточен. Для функции доставки необходимо отдельно рассматривать:

- duplicate ingress;
- process/VPS restart;
- network loss;
- rate limit;
- retryable failure;
- terminal failure;
- ambiguous send (`uncertain`);
- adapter reconnect;
- media quota exhaustion;
- SQLite reopen/crash recovery.

## Приватность

MsgHub ретранслирует частные сообщения, поэтому содержимое текста и вложений не должно попадать в обычный диагностический вывод. Для диагностики предпочитать:

- canonical/event/message identifiers;
- platform/endpoint;
- timestamp;
- размер payload;
- MIME/type;
- delivery state;
- error class/code;
- retry/reconciliation state.

Полный payload разрешён только в явно включённом режиме диагностики с предупреждением о приватности и без секретов аутентификации.

## Зависимости и инфраструктура

Не добавлять инфраструктуру «на будущее». Для v0.1 исходно предполагаются:

- один небольшой Ubuntu VPS;
- SQLite в WAL mode;
- локальный ограниченный media cache;
- relay core;
- отдельный WhatsApp adapter process, если выбран Web-compatible transport.

Любая новая обязательная служба или тяжёлая runtime-зависимость требует измерения и объяснения, почему существующая схема не удовлетворяет требованиям.

## Работа с внешними платформами

Платформенные особенности документируются в адаптере и capability contract. Core не должен знать детали Telegram Bot API, WhatsApp Web, VK или MAX сверх платформонезависимого контракта.

Для неофициальных/обратно разработанных транспортов отдельно фиксировать:

- риск поломки после обновления платформы;
- login/session lifecycle;
- восстановление после logout;
- ambiguous send behavior;
- требования к версии библиотеки;
- процедуру безопасного обновления и rollback.

## Проверки до merge

До выбора стека в Issue #2 обязательных команд тестов нет. Минимум для документационных PR:

- проверить согласованность README, architecture, MVP и roadmap;
- проверить отсутствие секретов/runtime state;
- проверить, что Issue/PR scope не расширен случайно.

После выбора стека этот раздел должен быть обновлён конкретными командами и синхронизирован с `dilukhin/github-connector-knowledge/src/profiles/dilukhin__msghub.yaml`.
