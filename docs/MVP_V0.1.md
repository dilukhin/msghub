# MVP v0.1

## Objective

Prove that MsgHub can reliably bridge one ordinary Telegram group and one ordinary WhatsApp group on a small Ubuntu VPS without losing accepted messages across process restarts and without creating relay loops.

The architecture must already support more than two endpoints per logical room, but v0.1 validation uses exactly one Telegram endpoint and one WhatsApp endpoint.

## Required user-visible behavior

### Phase A — reliable text relay

A text message posted by a human participant in either connected group appears in the other group with clear sender attribution.

Required:

- Telegram -> WhatsApp text;
- WhatsApp -> Telegram text;
- sender display name attribution;
- durable delivery across service restart;
- persistent message ID mapping;
- no self-amplifying relay loop;
- bounded retry with visible terminal failure state;
- basic health/status diagnostics.

### Phase B — conversation structure

Required:

- native replies are preserved when both sides provide enough identifiers;
- fallback rendering is deterministic when a native reply cannot be preserved;
- duplicated inbound events are idempotently ignored.

### Phase C — common media

Required media classes:

- images;
- ordinary files/documents;
- voice/audio where practical with the selected transports.

Required media behavior:

- temporary local download only when needed for bridging;
- configurable per-file limit;
- bounded total cache;
- cleanup after all dependent deliveries become safe to remove;
- text relay remains operational when media intake is rejected because of storage limits.

Video may be enabled if it falls out naturally from the selected adapters, but large-video optimization is not a v0.1 gate.

## Out of scope for v0.1

These are intentionally deferred unless required to make the basic bridge correct:

- message edits;
- message deletion propagation;
- reactions;
- polls;
- stickers as first-class cross-platform objects;
- live locations;
- history import before MsgHub was connected;
- synchronization of group membership;
- synchronization of administrator roles/permissions;
- end-user identity merging across platforms;
- graphical administration UI;
- multi-node/high-availability deployment;
- permanent media archive;
- Redis, RabbitMQ, Kafka, or another external queue;
- PostgreSQL as an initial requirement.

## Reliability contract

MsgHub does not claim mathematically exact once-only delivery across independent third-party messaging platforms.

The v0.1 target is:

```text
durable ingress
+ persistent at-least-once processing
+ platform-aware idempotency/deduplication where possible
+ explicit uncertain state when remote acceptance is unknown
```

An event is considered accepted by MsgHub only after its canonical representation and required delivery records are committed to durable storage.

## Failure expectations

The bridge must recover sensibly from:

- relay-core restart;
- WhatsApp adapter restart;
- temporary network loss;
- Telegram API transient failure;
- WhatsApp reconnect/session churn;
- duplicate inbound event delivery;
- SQLite process interruption/crash recovery;
- destination platform rate limiting;
- full/near-full media cache.

A WhatsApp adapter failure must not corrupt the database or require rebuilding Telegram state.

## Deployment target

Initial production-like target:

```text
Ubuntu VPS
1-2 vCPU
1-2 GiB RAM
~20 GiB SSD preferred
```

The implementation should avoid a headless Chromium requirement if a stable enough browserless WhatsApp transport is validated, because memory/disk efficiency matters for the intended VPS class.

## Storage target

Text and metadata may be retained long-term.

Media is temporary and bounded. The initial operational target is approximately a 5 GiB soft media-cache budget on a 20 GiB VPS, with a separately configurable hard ceiling and sufficient reserved disk space for the operating system, database, logs, and upgrades.

Exact limits are configuration, not protocol constants.

## Acceptance test

v0.1 is complete when an operator can configure two real groups and pass an end-to-end test covering at least:

1. Telegram text -> WhatsApp;
2. WhatsApp text -> Telegram;
3. reply Telegram -> WhatsApp -> Telegram mapping;
4. reply WhatsApp -> Telegram -> WhatsApp mapping;
5. duplicate inbound event does not duplicate an outbound message;
6. a relay-created message received back from an adapter is not re-relayed;
7. restart with pending work resumes delivery;
8. transient send failure retries after restart;
9. an uncertain send is recorded distinctly from a known failure;
10. image/file relay with cache cleanup;
11. media quota exhaustion does not prevent ordinary text relay;
12. health output makes adapter/database/queue state diagnosable without exposing secrets.

## Definition of done

- architecture and wire/storage schemas used by implementation are versioned/documented;
- automated tests cover core routing, deduplication, message mapping, delivery-state transitions, and storage limits;
- real Telegram/WhatsApp smoke procedure is documented;
- systemd deployment and backup/restore procedure are documented;
- no secrets or WhatsApp session material are committed to the repository;
- a fresh small Ubuntu VPS can be brought up from documented steps.
