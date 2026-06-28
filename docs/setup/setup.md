# Setup & Usage

## Prerequisites

| Requirement | Version |
|---|---|
| Docker | 24+ |
| Docker Compose | V2 (`docker compose`) |
| Python | 3.11 (only needed for local non-Docker runs) |

---

## Quick start (Docker Compose)

```bash
git clone <repo-url>
cd classify_mahalli
docker compose up --build
```

This starts six containers:

| Container | Role |
|---|---|
| `zookeeper` | Kafka coordination |
| `kafka` | Apache Kafka 3.9.0 (port 29092 external) |
| `kafka-ui` | Web UI at http://localhost:8080 |
| `producer` | Sends synthetic CDC events |
| `cdc_filter` | Filters description changes |
| `spam_detector` | Classifies listings |

The spam detector loads the Harrier ONNX model on first startup (~30 s). Open the
Kafka UI at **http://localhost:8080** → Topics → `product.spam.detections` to see
results in real time.

### Run only the detector (skip synthetic producer)

```bash
docker compose up --build spam_detector cdc_filter kafka zookeeper
python scripts/ingest_export.py   # loads 500 real CDC records from export file
```

---

## Environment variables

All variables have defaults set in `docker-compose.yml`. Override with a `.env` file.

### CDC Filter

| Variable | Default | Description |
|---|---|---|
| `KAFKA_BOOTSTRAP_SERVERS` | `kafka:9092` | Kafka cluster address |
| `INPUT_TOPIC` | `prod.mysql.salla.products` | CDC source topic |
| `OUTPUT_TOPIC` | `product.description.changes` | Filtered output topic |

### Spam Detector

| Variable | Default | Description |
|---|---|---|
| `KAFKA_BOOTSTRAP_SERVERS` | `kafka:9092` | Kafka cluster address |
| `INPUT_TOPIC` | `product.description.changes` | Input topic |
| `OUTPUT_TOPIC` | `product.spam.detections` | Output topic |
| `SPAM_PERSISTENCE_PATH` | `/app/spam_detections.jsonl` | JSONL output path |
| `DB_HOST` | *(unset)* | MySQL host — persistence skipped if unset |

### Producer

| Variable | Default | Description |
|---|---|---|
| `PRODUCE_INTERVAL_MIN` | `0.02` | Min delay between messages (seconds) |
| `PRODUCE_INTERVAL_MAX` | `0.2` | Max delay between messages (seconds) |

---

## Running the validator

```bash
pip install -r requirements-ml.txt -r requirements-service.txt
python evaluate_xgboost.py
```

Classifies the first 100 rows of `test_data.csv` and writes:
- `false_positives_to_analyze.json`
- `false_negatives_to_analyze.json`

---

## Model artifacts

| File | Size | Used by |
|---|---|---|
| `artifacts/XGBoost/xgb_model_trained_5k_Harrier.json` | 1.17 MB | Kafka detector (default) |
| `[spam_pipeline]/xgb_model_text_embeddings.json` | — | Mage pipeline (text model) |
| `[spam_pipeline]/xgb_model_image_embeddings.json` | — | Mage pipeline (image model) |

The Harrier ONNX weights are downloaded from HuggingFace at build time via
`light-embed`. An internet connection is required on the first `docker compose build`.

---

## Switching the active model

Edit `services/spam_detector/configs/config.py`:

```python
DEFAULT_SCAM_MODEL = "Harrier_xgb"   # or "minilm_xgb"
```

Model paths and embedder configurations are in `configs/models.yaml`.
