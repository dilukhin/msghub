# MVP v0.1

## Цель

Доказать, что MsgHub может надёжно объединить одну обычную Telegram-группу и одну обычную WhatsApp-группу на небольшой Ubuntu VPS без потери уже принятых сообщений при перезапусках и без возникновения relay loop.

Архитектура при этом уже должна поддерживать более двух endpoint-ов внутри одного `LogicalRoom`, хотя v0.1 валидируется на паре Telegram + WhatsApp.

## Обязательное поведение

### Этап A — надёжный текстовый мост

Сообщение человека в любой из двух групп появляется в другой группе с понятным указанием автора.

Обязательно:

- Telegram -> WhatsApp text;
- WhatsApp -> Telegram text;
- sender attribution;
- долговечная доставка через restart;
- постоянное сопоставление message IDs;
- отсутствие самораскручивающегося relay loop;
- ограниченный retry с диагностируемым terminal failure;
- базовый health/status.

### Этап B — структура разговора

Обязательно:

- native replies сохраняются, когда обе платформы дают необходимые ID;
- fallback для невозможного native reply детерминирован;
- duplicate ingress обрабатывается идемпотентно;
- echo сообщений, созданных самим MsgHub, не запускает повторную маршрутизацию.

### Этап C — распространённое мультимедиа

Обязательные классы:

- изображения;
- обычные файлы/документы;
- voice/audio, если выбранные transports позволяют практичное отображение.

Обязательное поведение media:

- локальное скачивание только когда оно нужно для доставки;
- configurable per-file limit;
- ограниченный общий cache;
- cleanup после того, как все зависимые Delivery безопасны для удаления;
- text relay продолжает работать, если media intake отклонён из-за нехватки выделенного бюджета.

Видео допустимо, если поддержка естественно получается из выбранных адаптеров, но оптимизация больших видео не является gate v0.1.

## Вне scope v0.1

Если иное не требуется для корректности базового моста, откладываются:

- propagation edits;
- propagation deletes;
- reactions;
- polls;
- stickers как first-class cross-platform objects;
- live location;
- импорт истории до подключения MsgHub;
- синхронизация состава групп;
- синхронизация прав администраторов;
- объединение пользовательских identity между платформами;
- графический интерфейс администрирования;
- multi-node/high-availability deployment;
- постоянный media archive;
- Redis/RabbitMQ/Kafka и внешняя очередь;
- PostgreSQL как обязательное исходное хранилище.

## Контракт надёжности

MsgHub не заявляет математический exactly-once между независимыми сторонними платформами.

Цель v0.1:

```text
durable ingress
+ persistent at-least-once processing
+ platform-aware idempotency/deduplication where possible
+ explicit uncertain state when remote acceptance is unknown
```

Событие считается принятым только после commit канонического представления и всех требуемых delivery records.

## Ожидаемые отказы

Мост должен корректно восстанавливаться после:

- restart relay core;
- restart WhatsApp adapter;
- временной потери сети;
- transient Telegram API failure;
- WhatsApp reconnect/session churn;
- duplicate inbound event;
- SQLite process interruption/crash recovery;
- destination rate limiting;
- заполнения/почти заполнения media cache.

Отказ WhatsApp adapter не должен повреждать БД или требовать перестроения Telegram state.

## Целевая VPS

Ориентир production-like окружения:

```text
Ubuntu VPS
1–2 vCPU
1–2 GiB RAM
около 20 GiB SSD предпочтительно
```

Если browserless WhatsApp transport проходит spike, v0.1 не должна требовать headless Chromium, поскольку память и диск важны для дешёвой VPS.

## Хранение

Text/metadata могут храниться долго.

Media временно и ограниченно. Исходный ориентир — около 5 GiB soft media-cache budget на VPS с диском около 20 GiB, отдельный hard ceiling и зарезервированное место для ОС, БД, логов и обновлений.

Точные значения являются настройками.

## Приёмочная проверка

v0.1 завершена, когда оператор может настроить две реальные группы и пройти как минимум следующие сценарии:

1. Telegram text -> WhatsApp;
2. WhatsApp text -> Telegram;
3. reply Telegram -> WhatsApp с корректным mapping;
4. reply WhatsApp -> Telegram с корректным mapping;
5. duplicate inbound event не создаёт duplicate outbound message;
6. сообщение, созданное relay и увиденное обратно адаптером, не ретранслируется повторно;
7. restart с pending work возобновляет доставку;
8. transient send failure переживает restart и корректно retry-ится;
9. ambiguous send записывается как `uncertain`, а не как известный failure;
10. image/file relay работает с последующим cleanup;
11. media quota exhaustion не блокирует обычный text relay;
12. health output позволяет диагностировать adapters/database/queue без раскрытия секретов и private payload.

## Definition of done

- архитектура и используемые wire/storage schemas документированы и версионируются;
- automated tests покрывают routing, deduplication, message mapping, delivery states и storage limits;
- documented real Telegram/WhatsApp smoke procedure;
- documented systemd deployment и backup/restore;
- секреты и WhatsApp session material отсутствуют в Git;
- свежая небольшая Ubuntu VPS может быть развёрнута по документации;
- вся человекочитаемая часть проекта соответствует русскому языку проекта.
