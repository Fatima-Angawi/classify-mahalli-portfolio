# 008. Mage.ai pipeline for Salla production

**Date:** June 2026

## Context

The spam classifier was initially designed as a Kafka-native streaming service: a CDC
filter and a detector service, both running as Docker containers. After the Kafka
pipeline was built and validated, Salla requested the production ML inference be
delivered as a **mage.ai pipeline** — Salla uses mage.ai as their production ML
orchestration platform and wanted the classifier to plug into their existing
infrastructure rather than run a separate Kafka cluster.

## Alternatives considered

- **Kafka streaming (original design)** — CDC filter + spam detector as microservices;
  output via Kafka → ClickHouse
- **Mage.ai pipeline** — block-based pipeline on Salla's mage.ai platform; output to
  MySQL

## Decision

**Mage.ai pipeline** for Salla's production environment. The Kafka pipeline is
retained as the self-contained development and testing environment. The mage.ai
pipeline lives in `[spam_pipeline]/` and is what Salla runs in production.

## Decision matrix

| Criteria | Kafka streaming | Mage.ai pipeline |
|---|---|---|
| Infrastructure fit | Good for CDC-native systems | Required — Salla's prod ML platform |
| Operational ownership | Self-managed | Managed by Salla's platform team |
| Deployment complexity | Requires Kafka cluster | Plugs into existing Salla infra |
| Monitoring | Custom monitor service | Mage.ai built-in monitoring |
| Scheduling | Consumer group always-on | Mage.ai trigger configuration |

## Pipeline structure

```
embeddings_loader → batch_embeddings → xgboost → predictions
```

| Block | What it does |
|---|---|
| `embeddings_loader` | Loads text + image embeddings from Salla's data source |
| `batch_embeddings` | Batches 256 items; passes both embedding arrays forward |
| `xgboost` | Runs two XGBoost models; blends `0.3 × text + 0.7 × image`; thresholds 0.7 / 0.4 |
| `predictions` | Exports results to MySQL |

## Consequences

Gained: direct integration with Salla's production ML platform — no additional
infrastructure on their side. Mage.ai provides built-in run history, pipeline
monitoring, and trigger scheduling. The pipeline-as-code structure is version-controlled.

Given up: the Kafka pipeline's real-time, event-driven nature is replaced by a batch
pipeline — latency characteristics differ. Two parallel implementations of the same
classifier logic must be kept in sync as the model evolves.
