# MsgHub

MsgHub is a self-hosted message relay for linking chats and groups across different messaging platforms into one logical conversation.

A message received in one connected endpoint is normalized, persisted, routed, and delivered to the other endpoints while preserving sender attribution, replies, attachments, and message relationships where the destination platform allows it.

## Initial scope

The first target is a bidirectional bridge:

```text
Telegram group <-> MsgHub <-> WhatsApp group
```

Planned adapters include VK, MAX, and other messaging platforms.

## Design goals

- platform-independent relay core;
- arbitrary number of platform endpoints in one logical room;
- durable ingress and delivery queue;
- loop prevention and deduplication;
- reply/message mapping between platforms;
- bounded temporary media storage instead of a permanent media archive;
- adapter capability negotiation for platform differences;
- recoverable operation on a small Ubuntu VPS;
- replaceable WhatsApp integration isolated from the relay core.

## Proposed architecture

```text
Platform APIs / clients
        |
        v
+-------------------+
| Platform adapters |
+---------+---------+
          |
          v
+-------------------+
|     Relay Core    |
| normalize         |
| route             |
| deduplicate       |
| map replies       |
+---------+---------+
          |
          v
+-------------------+
| Durability layer  |
| SQLite            |
| deliveries        |
| message mappings  |
| bounded media     |
+-------------------+
```

The initial deployment model is a modular monolith for the stable core and official API adapters, with the WhatsApp Web compatibility adapter isolated as a separate process/failure domain.

## Status

Design / pre-alpha. No production implementation exists yet.

The initial architecture and MVP contract are documented under [`docs/`](docs/).
