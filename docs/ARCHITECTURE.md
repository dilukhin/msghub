# Архитектура

## Назначение

MsgHub объединяет чаты разных мессенджеров в единый логический разговор и не привязывает relay core к API конкретной платформы.

Первая реализация ориентирована на Telegram и WhatsApp. Архитектура должна позволять добавлять VK, MAX и другие платформы без переделки маршрутизации и долговременного состояния.

## Базовые сущности

### LogicalRoom

Платформонезависимый логический разговор, управляемый MsgHub. В одном `LogicalRoom` находится один или несколько endpoint-ов.

Пример:

```text
logical room: family
  - telegram / group A
  - whatsapp / group B
  - max / group C
```

### Endpoint

Конкретный чат, группа или канал одной платформы. В исходной модели endpoint принадлежит ровно одному `LogicalRoom`.

### CanonicalEvent

Нормализованное событие, которое создаёт адаптер и потребляет relay core. Нативные payload-ы Telegram/WhatsApp/VK/MAX не должны проникать в маршрутизацию.

Первый обязательный тип:

```text
message.created
```

На будущее резервируются:

```text
message.edited
message.deleted
reaction.added
reaction.removed
```

Минимальный состав message event:

```text
event_id
logical_room_id
source.platform
source.endpoint_id
source.message_id
author.platform_user_id
author.display_name
timestamp
message.text
message.reply_to
message.attachments[]
metadata
```

`metadata` может содержать ограниченные adapter-specific диагностические сведения. Корректность маршрутизации не должна зависеть от недокументированных платформенных полей.

### Delivery

Долговечная задача материализовать один `CanonicalEvent` на одном целевом endpoint.

Начальная машина состояний:

```text
pending -> sending -> delivered
                   -> retry
                   -> uncertain
                   -> failed
```

`uncertain` принципиально отличается от `failed`: транспорт оборвался, а доказательств того, приняла ли удалённая платформа сообщение, недостаточно. Слепой retry может создать дубликат, поэтому требуется reconciliation либо явно принятая политика.

### MessageLink

Связывает каноническое сообщение с его платформенными копиями.

```text
canonical message X
  telegram -> message 931
  whatsapp -> message ABC
  max      -> message 71291
```

`MessageLink` нужен для:

- reply mapping;
- дедупликации;
- защиты от циклов;
- будущих edit/delete операций.

### MediaObject

Временное локальное представление вложения, необходимое для передачи между платформами. MsgHub не является постоянным архивом мультимедиа.

## Компонентная схема

```text
+---------------------------+
| Platform APIs / clients   |
+-------------+-------------+
              |
              v
+---------------------------+
| Platform adapters         |
| receive / normalize       |
| send / map result         |
| media I/O                 |
| capabilities / health     |
+-------------+-------------+
              |
              v
+---------------------------+
| Relay Core                |
| room resolution           |
| deduplication             |
| loop prevention           |
| routing                   |
| reply resolution          |
| delivery policy           |
+-------------+-------------+
              |
              v
+---------------------------+
| Durability layer          |
| SQLite                    |
| event log                 |
| delivery queue            |
| message links             |
| adapter state             |
+-------------+-------------+
              |
              v
+---------------------------+
| Bounded media cache       |
+---------------------------+
```

## Граница адаптера

Relay core работает с небольшим общим контрактом, а не с объектами SDK конкретного мессенджера.

Концептуально:

```text
start()
stop()
receive() -> CanonicalEvent
send_message(event, target) -> SendResult
download_media(ref) -> MediaObject
upload_media(media, target) -> RemoteMediaRef
capabilities() -> CapabilitySet
health() -> AdapterHealth
```

Точный программный интерфейс определяется после выбора языка реализации в Issue #2.

## Входящий поток

```text
1. Адаптер получает платформенное событие.
2. Адаптер нормализует его в CanonicalEvent.
3. Core определяет source Endpoint и LogicalRoom.
4. Core проверяет идентичность source и deduplication constraints.
5. Event долговечно записывается.
6. Для каждого допустимого target Endpoint создаётся Delivery.
7. Транзакция фиксируется.
8. Delivery workers начинают обработку pending work.
```

Для фиксации ingress не требуется успешная внешняя отправка.

Событие считается принятым MsgHub только после commit его канонического состояния и связанных delivery records.

## Исходящая доставка

```text
1. Worker атомарно забирает pending/retry Delivery.
2. При наличии reply разрешается MessageLink родителя.
3. Формируется представление автора для целевой платформы.
4. При необходимости материализуется media.
5. Адаптер выполняет send.
6. Сохраняется remote message ID и итоговый state.
7. Media освобождается только когда все зависимые Delivery безопасны для удаления.
```

## Защита от циклов и дедупликация

Защита от relay loop является требованием корректности.

Нельзя полагаться на:

- скрытые текстовые маркеры;
- сравнение текста сообщений;
- только `from_self` или аналогичный флаг платформы.

Нужны как минимум два постоянных механизма:

1. уникальность `(platform, endpoint_id, source_message_id)` для принятых нативных сообщений;
2. `MessageLink`/`Delivery` records для сообщений, созданных MsgHub на целевых платформах.

Нативный `from_self` допускается как дополнительная ранняя оптимизация.

## Replies

Если платформа сообщает reply на локальное сообщение, адаптер передаёт ID локального родителя. Core:

```text
local replied-to ID
  -> MessageLink
  -> canonical parent
  -> destination-native parent ID
```

Если native reply на целевой платформе невозможен, используется детерминированный fallback: цитата/контекст по правилам capability policy.

## Возможности платформ

Каждый адаптер объявляет `CapabilitySet`, например:

```text
send_text
send_image
send_video
send_file
send_audio
reply
edit
delete
reaction
mention
poll
location
```

При отсутствии capability core применяет явную per-room policy: degrade, skip или fail. Неявное поведение недопустимо.

## Долговременное состояние

Для v0.1 используется SQLite в WAL mode.

Логические таблицы:

- `logical_rooms`;
- `endpoints`;
- `events`;
- `deliveries`;
- `message_links`;
- `identities` — необязательно для первого MVP;
- `adapter_state`;
- `media_objects`.

Очередь доставки хранится в той же БД, чтобы accepted ingress и outbound work фиксировались атомарно.

Переход на PostgreSQL возможен позже при доказанной необходимости. Core contracts не должны зависеть от SQLite-specific поведения сильнее, чем требуется текущей реализации.

## Мультимедиа

Мультимедиа — временное состояние доставки.

Исходный ориентир:

```text
soft cache target: около 5 GiB
hard cache ceiling: configurable
successful delivery retention: около 24 h
failed/retry retention: дольше, например до 7 d
```

Точные значения — настройки.

Cleanup не должен удалять объект, пока он нужен хотя бы одной доставке в состоянии:

```text
pending
sending
retry
uncertain
```

Заполнение media cache не должно лишать SQLite возможности фиксировать текстовые сообщения. При hard pressure допустимо отказаться от нового крупного media, продолжая text relay.

## Изоляция WhatsApp

Для обычной пользовательской WhatsApp-группы v0.1 может потребоваться WhatsApp Web-совместимый transport вместо официального business-group API.

Такой transport считается:

- повышенно рискованным;
- заменяемым;
- потенциально чувствительным к изменениям WhatsApp;
- отдельной зоной отказа.

Предпочтительная граница:

```text
relay-core <-> versioned local IPC <-> whatsapp-adapter <-> WhatsApp transport
```

Будущий официальный адаптер должен уметь реализовать тот же логический контракт без переделки core routing/persistence.

## Модель развёртывания

Первый target — одна небольшая Ubuntu VPS.

```text
systemd
  msghub-core.service
  msghub-whatsapp.service

/etc/msghub/
  configuration
  secrets

/var/lib/msghub/
  database
  adapter state
  media cache
```

Reverse proxy нужен только если выбранные адаптеры требуют входящих HTTPS webhooks.

Ориентир ресурсов:

```text
1–2 vCPU
1–2 GiB RAM
около 20 GiB SSD
```

## Границы безопасности

- секреты и session state не попадают в Git;
- каждому адаптеру выдаются только необходимые credentials;
- diagnostics по умолчанию не содержат tokens, cookies, QR/session material и private payload;
- database/media/session files читаются только сервисным пользователем MsgHub;
- inbound webhook аутентифицируется, если платформа предоставляет соответствующий механизм;
- полное содержимое приватных сообщений не является штатной частью логов.

## Архитектурные инварианты

1. Platform-native objects не являются core data model.
2. Ingress долговечен до признания события принятым.
3. Retry state переживает restart.
4. Loop prevention переживает restart.
5. `uncertain` не приравнивается к обычному retry.
6. Media storage ограничено.
7. Отказ одного адаптера не должен повреждать состояние остальных endpoint-ов/room-ов.
8. Новая платформа не требует нового попарного Telegram-to-X router.
9. Приватные payload-ы не являются штатным диагностическим выводом.
10. Архитектура v0.1 должна оставаться пригодной для дешёвой VPS.

## Открытые решения до реализации

- язык и runtime;
- точная схема `CanonicalEvent v1` и versioning;
- конкретный local IPC;
- SQLite worker/locking strategy;
- retry/reconciliation policy по адаптерам;
- Telegram receive mode: long polling или webhook;
- библиотека/transport WhatsApp после feasibility spike;
- config format и secret injection;
- observability/health contract;
- конкретные команды тестов и CI jobs.
