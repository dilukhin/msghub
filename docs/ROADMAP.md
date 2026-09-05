# Дорожная карта

Дорожная карта выстроена вокруг снижения рисков. Главная ранняя неопределённость — не маршрутизация, а практическое поведение и эксплуатационная стабильность WhatsApp endpoint для обычной пользовательской группы.

## Фаза 0 — зафиксировать core contract

Цель: убрать архитектурную неоднозначность до реализации.

Результаты:

- `CanonicalEvent v1` schema;
- endpoint / logical-room model;
- delivery state machine;
- MessageLink и reply-resolution rules;
- capability model и degradation policy;
- media lifecycle/quota contract;
- config/secrets model;
- observability/health contract;
- решение по языку/runtime;
- требования к IPC для изолированных adapters.

Gate: core behavior можно полноценно тестировать без реальных мессенджеров.

GitHub: Issue #2.

## Фаза 1 — feasibility spikes адаптеров

### WhatsApp spike

Проверить транспорт для обычной WhatsApp-группы до того, как production-код начнёт зависеть от конкретной библиотеки.

Нужно получить фактические ответы:

- pairing/login flow;
- unattended restart;
- session persistence и backup/recovery;
- group receive/send semantics;
- стабильные group/message/user IDs;
- self-echo/from-self behavior;
- replies/quoted IDs;
- media download/upload;
- duplicate events;
- reconnect behavior;
- ambiguous send при потере соединения;
- retryable/terminal errors;
- CPU/RAM/disk/network footprint;
- риск protocol/library breakage и процедура recovery.

Spike-код может быть одноразовым. Итог — документированный transport contract и решение go/no-go.

GitHub: Issue #3.

### Telegram spike

Подтвердить нужное MsgHub поведение Telegram Bot API:

- bot permissions/privacy;
- long polling vs webhook;
- стабильные IDs;
- sender attribution;
- replies;
- files/media;
- duplicate/update delivery;
- rate/error handling;
- self-authored bot message behavior;
- restart semantics;
- adapter capabilities/limits.

GitHub: Issue #4.

Gate фазы: оба транспорта имеют evidence-backed contract, достаточный для реализации MVP.

## Фаза 2 — платформонезависимое ядро

Реализовать и протестировать без реальных messaging services:

- config model;
- `LogicalRoom`/`Endpoint` registry;
- `CanonicalEvent v1`;
- SQLite schema/migrations;
- transactional ingress;
- durable Delivery queue;
- delivery state machine;
- retry scheduling;
- deduplication/loop-prevention records;
- MessageLink;
- reply resolution;
- adapter capability interface;
- MediaObject lifecycle/quota accounting;
- health/diagnostic snapshot.

Gate: deterministic tests на fake adapters проходят restart, duplicate, retry, terminal failure и uncertain cases.

GitHub: Issue #5.

## Фаза 3 — Telegram adapter

Реализовать production Telegram adapter против зафиксированного core contract.

Gate: реальная Telegram-группа умеет отдавать события в core и принимать сообщения из core, при этом core tests не зависят от Telegram.

GitHub: Issue #6.

## Фаза 4 — изолированный WhatsApp adapter

Реализовать выбранный transport отдельным процессом/зоной отказа с versioned local IPC.

Gate: restart/reconnect адаптера не требует restart relay core и не повреждает database state.

GitHub: Issue #7.

## Фаза 5 — сквозной текстовый мост

Соединить одну Telegram-группу и одну WhatsApp-группу.

Обязательные gates:

- bidirectional text;
- sender attribution;
- loop prevention;
- deduplication;
- restart recovery;
- reply mapping;
- diagnostics;
- controlled handling of uncertain sends.

Это первый полезный release candidate.

GitHub: Issue #8.

## Фаза 6 — ограниченное мультимедиа

Добавить изображения, документы и voice/audio в рамках строгого media-cache lifecycle.

Обязательные gates:

- per-file limit;
- total-cache quota;
- safe cleanup;
- restart recovery;
- media, нужное live delivery, не удаляется;
- text path продолжает работать при media pressure.

GitHub: Issue #9.

## Фаза 7 — эксплуатация и v0.1

- systemd units;
- Ubuntu installation guide;
- secret/session provisioning;
- backup/restore;
- database maintenance;
- log rotation;
- health checks;
- recovery runbook;
- real-group acceptance test;
- release/versioning policy.

GitHub: Issue #10.

## После v0.1

После стабилизации первой версии:

- MAX adapter;
- VK adapter;
- edits/deletes;
- reactions;
- richer media/sticker policy;
- cross-platform identity mapping;
- несколько LogicalRoom и administration tooling;
- optional PostgreSQL/object storage, только если фактический scale этого потребует.

## Правила планирования

- Архитектурные решения отделять от transport experiments.
- Не писать pairwise `Telegram -> WhatsApp` routing: все маршруты проходят через canonical core.
- Не добавлять инфраструктуру только потому, что она «обычно используется»; SQLite и bounded local media предпочтительны до появления измерений против них.
- Неофициальные/reverse-engineered transports считать заменяемыми dependencies с отдельным operational risk.
- Reliability feature считается реализованной только если проверена на failure/restart, а не только на happy path.
- Каждую задачу удерживать в scope соответствующего Issue; новые крупные требования оформлять отдельно.
