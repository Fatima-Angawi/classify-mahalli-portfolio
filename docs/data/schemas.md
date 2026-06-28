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

## Training / evaluation DataFrame

| Column | Type | Description |
|---|---|---|
| `id` | int | Product ID |
| `name` | str | Product name (Arabic / mixed) |
| `description` | str | HTML or plain text description |
| `image_url` | str | Product image URL |
| `store_id` | int | Salla store identifier |
| `brand` | str | Brand name |
| `custom_id` | str | Merchant-assigned custom ID |
| `label` | int | Binary spam label (1 = spam, 0 = not spam) |
| `model_used` | str | Which model produced this row's embedding |
| `image_embedding` | list[float] | Image embedding vector (SigLIP) |
| `image_embedding_dim` | int | Dimensionality of image embedding |
| `has_image` | bool | Whether a product image was available |
| `image_model_x` / `image_model_y` | str | Image encoder identifier |
| `text_embedding` | list[float] | Text embedding vector (Harrier) |
| `text_embedding_dim` | int | Dimensionality of text embedding |
| `text_model` | str | Text encoder identifier |
| `llm_label` | str | Ground-truth label from GPT-4.1-mini (`spam` / `not_spam`) |
| `text_cluster` | int | KMeans cluster ID on text embeddings |
| `image_cluster` | int | KMeans cluster ID on image embeddings |
| `store_id_x_x` / `store_id_y_x` / … | int | Store ID join artifacts from merging product and store scores tables |

---

## Store scores table

One row per store. Used to flag suspicious stores and enrich product-level features.

| Column | Type | Description |
|---|---|---|
| `store_id` | int | Salla store identifier |
| `total_products` | int | Total product listings |
| `spam_count` | int | Number of spam listings |
| `non_spam_count` | int | `total_products - spam_count` |
| `spam_rate` | float | `spam_count / total_products × 100` |
| `is_suspicious` | bool | `spam_rate ≥ 40% AND spam_count ≥ 10 AND total_products ≥ 20` |
| `store_score` | float | `spam_rate × spam_count` — composite risk score |

Sorted: suspicious stores first, then by `store_score` descending.

---

## Store context enrichment on product records

Each product row is enriched with two columns after joining the store scores:

| Column | Type | Values |
|---|---|---|
| `is_suspicious_store` | bool | Whether the product's store is flagged |
| `store_context` | str | `"spam_from_suspicious_store"` / `"spam_from_clean_store"` / `"normal"` |

```
label == spam AND is_suspicious_store      →  "spam_from_suspicious_store"
label == spam AND NOT is_suspicious_store  →  "spam_from_clean_store"
otherwise                                  →  "normal"
```

`store_context` is used as a feature signal — spam from a store with a high
historical spam rate is a stronger signal than an isolated listing from an
otherwise clean store.

`training_data.csv` columns: `name`, `description`, `spam_status`, `store_id`.
