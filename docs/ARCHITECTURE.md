# Architecture

## Purpose

MsgHub connects chats from different messaging platforms into a single logical conversation without making the relay core depend on any one platform API.

The initial implementation targets Telegram and WhatsApp, while the architecture must allow VK, MAX, and additional adapters to be added without redesigning routing or persistence.

## Core concepts

### LogicalRoom

A platform-independent conversation managed by MsgHub. A room contains one or more platform endpoints.

Example:

```text
logical room: family
  - telegram / group A
  - whatsapp / group B
  - max / group C
```

### Endpoint

A concrete chat/group/channel on one platform. An endpoint belongs to exactly one LogicalRoom in the initial design.

### CanonicalEvent

A normalized event produced by an adapter and consumed by the relay core. Platform-native payloads must not leak into routing logic.

Initial event kinds:

- `message.created`
- reserved for later: `message.edited`, `message.deleted`, reactions and other platform events.

A message event contains at least:

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

`metadata` may retain adapter-specific diagnostic data, but routing correctness must not depend on undocumented platform fields.

### Delivery

A durable attempt to materialize one CanonicalEvent on one destination endpoint.

Initial states:

```text
pending -> sending -> delivered
                   -> retry
                   -> uncertain
                   -> failed
```

`uncertain` is distinct from `failed`: it means the transport connection ended without enough evidence to know whether the remote platform accepted the message. Blind retry from this state can create duplicates and therefore requires adapter-specific reconciliation or an explicit policy.

### MessageLink

Maps one canonical message to its platform-native copies. It is required for reply reconstruction, deduplication, edits/deletes later, and loop prevention.

```text
canonical_message X
  telegram -> message 931
  whatsapp -> message ABC
  max      -> message 71291
```

### MediaObject

A temporary local representation of an attachment used to bridge platforms. MsgHub is not intended to become a permanent media archive.

## Component model

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

## Adapter boundary

The relay core works against a small adapter contract rather than platform SDK objects.

Conceptually:

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

The exact programming-language interface is intentionally deferred until the implementation language is chosen.

## Ingress flow

```text
1. Adapter receives a platform event.
2. Adapter normalizes it to CanonicalEvent.
3. Core resolves the source Endpoint and LogicalRoom.
4. Core checks source identity/deduplication constraints.
5. Event is persisted transactionally.
6. One Delivery is created for every eligible destination endpoint.
7. Transaction commits.
8. Delivery workers process pending work.
```

No external send should be required for the ingress transaction to commit.

## Delivery flow

```text
1. Claim a pending/retry Delivery.
2. Resolve reply target through MessageLink if present.
3. Render sender attribution for the destination platform.
4. Materialize required media.
5. Send through destination adapter.
6. Persist remote message ID and mark delivered.
7. Release media when all dependent deliveries are terminal and retention policy permits cleanup.
```

## Loop prevention and deduplication

Loop prevention is a correctness requirement, not a presentation trick.

The system must not depend on hidden text markers or comparing rendered message text.

Two persistent mechanisms are required:

1. uniqueness of `(platform, endpoint_id, source_message_id)` for ingested native messages;
2. MessageLink/Delivery records for messages created by MsgHub on destination platforms.

Adapters may additionally suppress events identified by a native `from_self`/equivalent flag, but that is only an optimization.

## Replies

When a platform reports a reply to one of its local messages, the adapter exposes the local replied-to message ID. The core resolves that ID through MessageLink to the canonical parent and then selects the corresponding destination-native parent ID, if one exists.

If a destination cannot preserve a native reply, the adapter may fall back to quoted/context text according to policy.

## Capability negotiation

Platforms do not expose identical semantics. Each adapter declares supported capabilities, for example:

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

The core applies per-room policy when a capability is missing: degrade, skip, or fail the delivery. MVP policy must be explicit rather than accidental.

## Persistence

SQLite in WAL mode is the initial persistence engine.

Expected logical tables:

- `logical_rooms`
- `endpoints`
- `events`
- `deliveries`
- `message_links`
- `identities` (optional for MVP)
- `adapter_state`
- `media_objects`

The delivery queue is stored in the same database so an accepted ingress event and its outbound work can be committed atomically.

PostgreSQL may replace SQLite later if scale requires it; routing and adapter contracts must not depend on SQLite-specific behavior.

## Media storage

Media is temporary delivery state, not the authoritative conversation archive.

Initial policy target:

```text
soft cache target: 5 GiB
hard cache ceiling: configurable
successful delivery retention: about 24 h
failed/retry retention: longer, e.g. up to 7 d
```

Exact defaults are implementation decisions and must remain configurable.

Cleanup must never remove media still required by a `pending`, `sending`, `retry`, or `uncertain` delivery.

Disk exhaustion must degrade media delivery without preventing the core from persisting and forwarding ordinary text messages where possible.

## WhatsApp isolation

The initial bridge for ordinary WhatsApp user groups may require a WhatsApp Web-compatible integration rather than an official business-group API.

This dependency is treated as replaceable and higher-risk. It should run in a separate process/failure domain from the stable relay core.

```text
relay-core <-> local IPC <-> whatsapp-adapter <-> WhatsApp transport
```

A future official adapter should be able to implement the same logical contract without changes to routing or persistence.

## Deployment model

Initial target: one small Ubuntu VPS.

```text
systemd
  relay-core.service
  relay-whatsapp.service

/etc/msghub/
  configuration
  secrets (permissions restricted)

/var/lib/msghub/
  database
  adapter state
  media cache
```

A reverse proxy is optional and only required when selected adapters use inbound HTTPS webhooks.

## Security boundaries

- credentials and session state are never committed to Git;
- platform tokens and WhatsApp session material are treated as secrets;
- adapters receive only the credentials they need;
- diagnostics must redact tokens, cookies, QR/session material and private payloads by default;
- media cache and database files should be readable only by the MsgHub service account;
- all inbound webhook endpoints must authenticate platform requests where the platform supports it.

## Architectural invariants

1. Platform-native objects do not become the core data model.
2. Ingress is durably recorded before outbound delivery is considered complete.
3. Delivery retry is persistent across process/VPS restarts.
4. Loop prevention survives restarts.
5. `uncertain` delivery is not silently treated as an ordinary retry.
6. Media storage is bounded.
7. One failing adapter must not corrupt or block unrelated room state.
8. Adding a new platform must not require a new pairwise Telegram-to-X router.

## Open decisions before implementation

- implementation language and runtime;
- concrete local IPC between core and isolated adapters;
- exact CanonicalEvent schema and versioning rules;
- SQLite locking/worker strategy;
- retry/reconciliation policy per adapter;
- initial Telegram receive mode (long polling vs webhook);
- exact WhatsApp integration library after a dedicated feasibility spike;
- configuration format and secret injection mechanism;
- observability contract and operator commands.
