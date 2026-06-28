# classify_mahalli — Product Spam Detection System

> The production code for this project is under a client NDA and cannot be shared publicly.
> This repository contains the system documentation I wrote during the project — architecture,
> design decisions, data schemas, evaluation results, and setup notes.

---

## Project summary

I built a real-time content moderation pipeline for **Salla**, one of the largest
e-commerce platforms in Saudi Arabia. The system automatically classifies product
listings as spam or legitimate and routes each to one of three outcomes:
`AUTO_REMOVE`, `HUMAN_REVIEW`, or `CLEAR`.

The pipeline consumes product update events from a Kafka CDC stream, runs a multimodal
XGBoost classifier (text + image embeddings), and publishes decisions to downstream
systems. Salla runs the classifier in production as a mage.ai pipeline integrated
into their ML platform.

**Key results:**
- Weighted Average fusion (0.3 text + 0.7 image): **F1 93.87%, Precision 97.45%**
- Image embeddings alone outperform text alone: F1 0.888 vs 0.607
- Iterated through three model architectures (AraBERT → MiniLM → Harrier + XGBoost)
  driven by a 150 msg/s throughput target

---

## Documentation

| Section | What's inside |
|---|---|
| [Architecture & System Overview](docs/architecture/overview.md) | How the system is built, how data flows, Kafka topics, mage.ai pipeline |
| [Setup & Usage](docs/setup/setup.md) | How to run the full stack locally with Docker Compose |
| [Data Schemas](docs/data/schemas.md) | Every data contract: Kafka messages, Pydantic models, output formats |
| [Results & Evaluation](docs/results/results.md) | Model benchmarks, fusion strategy comparison, error analysis |
| [Design Decisions](docs/decisions/INDEX.md) | 8 decisions covering model selection, pipeline architecture, and policy choices |

---

## Tech stack

- **ML:** XGBoost, Harrier (microsoft/harrier-oss-v1-270m), SigLIP (google/siglip2-base-patch16-224), ONNX runtime
- **Pipeline:** Apache Kafka, Debezium CDC, mage.ai
- **Labeling:** GPT-4.1-mini via OpenAI Batch API, weak supervision
- **Infrastructure:** Docker Compose, Python 3.11, ClickHouse, MySQL
- **Arabic NLP:** pyarabic, custom normalisation pipeline
