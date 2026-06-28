# 004. CDC filter as an isolated service

**Date:** May 2026

## Context

Every product update in MySQL — price changes, stock changes, view count increments —
generates a CDC event. Running the embedding step on every event would waste compute
on updates that carry no new content. The question was whether to filter
description-change events inside the detector or split it into a dedicated service.

## Alternatives considered

- **Filter inside the spam detector** — one service reads the raw CDC topic, skips
  irrelevant events internally
- **CDC filter as a separate service** — a lightweight service reads the raw topic and
  only forwards relevant events to a second topic

## Decision

**Separate CDC filter service.** It runs as its own Docker container, consuming
`prod.mysql.salla.products` and producing to `product.description.changes`. The spam
detector never sees a raw CDC event.

## Decision matrix

| Criteria | Filter inside detector | Separate filter service |
|---|---|---|
| Throughput isolation | Poor — detector wakes up per event | Good — embedder only runs on filtered events |
| Operational independence | Poor — coupled scaling | Good — each service scales independently |
| Topic reuse | None | `product.description.changes` usable by other consumers |
| Code complexity | Simpler — one fewer service | Two services, two Dockerfiles |
| Debuggability | Harder to inspect pre-filter volume | Filter lag visible separately in Kafka UI |

## Consequences

Gained: the embedding step only runs when a description actually changed —
proportional to content velocity, not total write volume. The intermediate topic can
be consumed by other downstream systems (e.g. search indexing). Filter and detector
can be restarted, scaled, or upgraded independently.

Given up: an extra Kafka consumer group and Docker container to operate.
