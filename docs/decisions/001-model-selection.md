# 001. Model selection: Harrier + XGBoost

**Date:** May 2026

## Context

The system needed to classify Arabic product listings in real time on a Kafka pipeline
with a throughput target of 150 msg/s. Two constraints drove the model search:
accuracy high enough for trust & safety, and inference fast enough to keep up with the
CDC event rate.

## Alternatives considered

- **AraBERT fine-tuned end-to-end** (`aubmindlab/bert-base-arabertv02`)
- **MiniLM + XGBoost** (`paraphrase-multilingual-MiniLM-L12-v2`)
- **Harrier + XGBoost** (`microsoft/harrier-oss-v1-270m`)
- **TF-IDF + XGBoost** (baseline)
- **multilingual-e5-small + XGBoost** (briefly tested)

## Decision

**Harrier + XGBoost.** Harrier was specifically chosen because it matches the embedding
infrastructure already deployed by Salla in their production environment — constraining
the search to what is already operationally available.

## Decision matrix

| Criteria | AraBERT | MiniLM + XGB | Harrier + XGB |
|---|---|---|---|
| F1 Score | 0.93 | 0.9839 | 0.93+ (fusion) |
| ROC-AUC | 0.9899 | — | 0.982 (image) |
| Inference latency | High — embedding bottleneck | 39–93 ms total | Lower (ONNX INT8) |
| Throughput | Below 150 msg/s target | ~5 msg/s | Target achievable |
| Deployment fit | Poor — model too large | Moderate | Matches Salla infra |
| Arabic quality | Excellent | Good | Good |

## Consequences

Gained: ONNX INT8 quantization via `light-embed` reduces latency and memory vs
AraBERT. XGBoost head is fast (~1 ms), interpretable, and easy to retrain independently
of the encoder.

Given up: AraBERT's purpose-built Arabic pre-training. MiniLM's F1 of 0.9839 looks
higher but was measured on an earlier dataset without image features — the numbers
are not directly comparable.
