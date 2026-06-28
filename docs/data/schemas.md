# Data Schemas

Every data contract in the pipeline: Kafka message formats, internal models, and
persistence output.

---

## Kafka: `prod.mysql.salla.products`

Raw CDC events from Debezium (or the synthetic producer in dev).

```json
{
  "before": {
    "id": 12345,
    "name": "string",
    "description": "string | null",
    "status": "string",
    "quantity": 10,
    "price": 99.99,
    "views": 100
  },
  "after": { "...same fields..." },
  "op": "u | c | d",
  "ts_ms": 1748700000000,
  "source": {
    "db": "salla",
    "table": "products",
    "connector": "mysql"
  }
}
```

`op`: `"u"` = update, `"c"` = create. Only `u` and `c` pass through the filter.
`before` is `null` for new records.

---

## Kafka: `product.description.changes`

Produced by the CDC Filter after stripping the CDC envelope down to what the
classifier needs.

```json
{
  "product_id": 12345,
  "name": "string",
  "description": "string"
}
```

`name` and `description` default to `""` when the source value was `null`.

---

## Kafka: `product.spam.detections`

Published by the Spam Detector as a ClickHouse upsert envelope.

```json
{
  "type": "upsert",
  "table": "spam_detections",
  "database": "salla",
  "service": "clickhouse_auto",
  "where_clause": { "product_id": 12345 },
  "data": {
    "product_id": 12345,
    "name": "string",
    "decision": "AUTO_REMOVE | HUMAN_REVIEW | CLEAR",
    "confidence": 0.87,
    "tier": "HIGH_RISK | MEDIUM_RISK | LOW_RISK | null",
    "source": "CLASSIFIER | RULE",
    "created_at": "2026-05-18T10:30:00.000000",
    "updated_at": "2026-05-18T10:30:00.000000"
  }
}
```

`confidence` is the XGBoost positive-class probability in [0, 1]. `tier` is `null`
when `source = "RULE"` (rule engine is currently disabled).

---

## Internal model: `ProductItem`

The unit passed between the consumer, moderation service, and scam model.

```python
@dataclass
class ProductItem:
    title: str
    description: str
```

Title and description are concatenated before embedding: `f"{title} {description}"`.

---

## Internal model: `ProductSnapshot`

Used when deserializing the `product.description.changes` message.

```python
class ProductSnapshot(BaseModel):
    id: int
    name: str = ""
    description: str = ""

    @field_validator("name", "description", mode="before")
    def coerce_none(cls, v) -> str:
        return "" if v is None else str(v)
```

---

## HTTP API models

These Pydantic models define a REST classify interface. The API endpoint is not wired
to a web server in the current codebase — the models exist for future integration.

```python
class Item(BaseModel):
    title: str = ""
    description: str = ""

class ClassifyRequest(BaseModel):
    items: list[Item]   # min length 1

class ItemResult(BaseModel):
    confidence: float   # [0.0, 1.0]
    decision: str       # AUTO_REMOVE | HUMAN_REVIEW | CLEAR
    source: str         # RULE | CLASSIFIER
    tier: str | None    # HIGH_RISK | MEDIUM_RISK | LOW_RISK
    rule_name: str | None

class ClassifyResponse(BaseModel):
    results: list[ItemResult]
```

---

## JSONL persistence output

Each detection written by the optional persistence layer:

```json
{
  "product_id": 12345,
  "name": "string",
  "decision": "AUTO_REMOVE | HUMAN_REVIEW | CLEAR",
  "confidence": 0.87,
  "tier": "HIGH_RISK | MEDIUM_RISK | LOW_RISK",
  "source": "CLASSIFIER | RULE",
  "created_at": "2026-05-18T10:30:00.000000",
  "updated_at": "2026-05-18T10:30:00.000000"
}
```

---

## Training / evaluation CSV

`test_data.csv` (1,130 rows) and `training_data.csv` (556,720 rows):

| Column | Type | Description |
|---|---|---|
| `id` | int | Product ID |
| `name` | str | Product name (Arabic / mixed) |
| `description` | str | HTML or plain text description |
| `spam_category` | str | Category from labeling pipeline (`restricted`, `iptv`, `digital_ambiguous`, …) |
| `llm_label` | str | Ground-truth label from GPT-4.1-mini (`spam` / `not_spam`) |
| `confidence` | float | LLM confidence score |
| `evidence` | str | Trigger span quoted by the LLM |

`training_data.csv` columns: `name`, `description`, `spam_status`, `store_id`.
