# 003. Kafka streaming over batch / REST API

**Date:** May 2026

## Context

Salla's product catalog updates arrive as Debezium CDC events from MySQL. The
classification system needed to process these events and write decisions to a downstream
store. The architecture had to fit Salla's existing infrastructure, which already uses
Kafka as the CDC transport.

## Alternatives considered

- **Kafka streaming** — native Kafka consumer/producer chain
- **Batch job** — periodic polling of MySQL or a dump file
- **REST API** — synchronous classify endpoint called by upstream systems

## Decision

**Kafka streaming.** The pipeline is a chain of Kafka consumers and producers: CDC
events flow from MySQL → Kafka → CDC filter → Kafka → spam detector → Kafka →
downstream.

## Decision matrix

| Criteria | Kafka streaming | Batch job | REST API |
|---|---|---|---|
| Latency | Low (seconds end-to-end) | High (minutes per batch) | Low (per-request) |
| Throughput | High — consumer batching amortises embedding cost | High | Limited by sync rate |
| Infrastructure fit | Excellent — Salla already runs Kafka | Poor — needs scheduler | Poor — requires caller integration |
| Decoupling | Good — filter and detector independent | Poor | Moderate |
| Fault tolerance | Good — Kafka offsets enable replay | Moderate | Low |

## Consequences

Gained: CDC filter and spam detector run as independent services — each can be scaled
or redeployed without touching the other. Kafka consumer offsets give free replay: if
the detector is down, messages queue and are processed when it recovers. Native fit
with Salla's existing Debezium → Kafka infrastructure.

Given up: a Kafka cluster must be running; local dev requires Zookeeper + Kafka
containers. Debugging a streaming pipeline requires monitoring topic lag and offsets,
which is harder than a batch script.
