# Roadmap

This roadmap is intentionally ordered around risk reduction. The first unknown is not routing logic; it is the practical behavior and operational stability of the WhatsApp endpoint intended for an ordinary user group.

## Phase 0 — freeze the design contract

Goal: remove architectural ambiguity before implementation.

Deliverables:

- CanonicalEvent v1 schema;
- endpoint / logical-room model;
- delivery state machine;
- message-link and reply-resolution rules;
- capability model and degradation policy;
- media lifecycle/quota contract;
- configuration/secrets model;
- observability/health contract;
- implementation language/runtime decision.

Exit gate: core behavior can be tested without any real messaging platform.

## Phase 1 — adapter feasibility spikes

### WhatsApp spike

Validate the intended ordinary-group transport before coupling production code to a specific library.

Questions:

- login/pairing flow and unattended restart behavior;
- group message receive/send semantics;
- stable native message identifiers;
- `from self`/echo behavior;
- replies and quoted message identifiers;
- media download/upload behavior;
- reconnect behavior around ambiguous sends;
- session persistence and backup implications;
- CPU/RAM/disk footprint on target VPS;
- upgrade/protocol-breakage risk.

The spike may be disposable code. Its output is a documented transport contract and go/no-go decision.

### Telegram spike

Confirm Bot API group behavior needed by MsgHub:

- receive mode;
- stable message/chat identifiers;
- replies;
- files;
- sender attribution;
- duplicate/update behavior;
- rate/error handling.

Exit gate: both adapters have evidence-backed contracts sufficient for MVP implementation.

## Phase 2 — platform-independent core

Implement and test without real messaging services:

- configuration model;
- LogicalRoom/Endpoint registry;
- CanonicalEvent;
- SQLite schema and migrations;
- transactional ingress;
- durable Delivery queue;
- delivery state machine;
- deduplication and loop-prevention records;
- MessageLink mapping;
- reply resolution;
- adapter capability interface;
- media-object lifecycle and quota accounting;
- health/diagnostic snapshot.

Exit gate: deterministic unit/integration tests using fake adapters pass restart, duplicate, retry and uncertain-delivery scenarios.

## Phase 3 — Telegram adapter

Implement Telegram against the frozen adapter contract.

Exit gate: real Telegram group can ingest and receive test messages while core tests remain platform-independent.

## Phase 4 — isolated WhatsApp adapter

Implement the selected WhatsApp transport as a separate process/failure domain with a versioned local IPC contract.

Exit gate: adapter restart/reconnect does not require relay-core restart or database repair.

## Phase 5 — end-to-end text bridge

Connect one Telegram group and one WhatsApp group.

Required gates:

- bidirectional text;
- attribution;
- loop prevention;
- deduplication;
- restart recovery;
- reply mapping;
- diagnostics;
- controlled handling of uncertain sends.

This is the first useful release candidate.

## Phase 6 — bounded media relay

Add images, documents and voice/audio under a strict media-cache lifecycle and disk budget.

Required gates:

- per-file limits;
- total-cache quota;
- safe cleanup;
- restart recovery;
- no deletion of media required by live deliveries;
- text path survives media-pressure conditions.

## Phase 7 — operations and v0.1 release

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

## Later

After v0.1 is stable:

- MAX adapter;
- VK adapter;
- message edits/deletes;
- reactions;
- richer media/sticker policies;
- cross-platform identity mapping;
- multiple logical rooms / administration tooling;
- optional PostgreSQL or external object storage if actual scale requires them.

## Principles for issue planning

- Keep architecture work separate from transport experiments.
- Do not implement pairwise `Telegram -> WhatsApp` routing; all routes pass through the canonical core.
- Do not add infrastructure merely because it is conventional. SQLite/local bounded media are preferred until measurements prove otherwise.
- Treat non-official or reverse-engineered transports as replaceable dependencies and explicit operational risk.
- Every reliability feature must be testable under restart/failure, not only on the happy path.
