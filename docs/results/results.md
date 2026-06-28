# Results & Evaluation

---

## Model progression

### Phase 1 — AraBERT fine-tuned end-to-end

**Model:** `aubmindlab/bert-base-arabertv02` as a sequence classifier

| Metric | Value |
|---|---|
| F1 Score | 0.93 |
| ROC-AUC | 0.9899 |

Strong Arabic classification accuracy, tracked with Weights & Biases. Rejected for
production because model size pushed throughput well below the 150 msg/s deployment
target on the Kafka pipeline.

---

### Phase 2 — MiniLM + XGBoost

**Model:** `paraphrase-multilingual-MiniLM-L12-v2` + XGBoost

| Metric | Value |
|---|---|
| F1 Score | 0.9839 |

**Inference latency samples:**

| Sample text | Embedding (ms) | XGBoost (ms) | Total (ms) |
|---|---|---|---|
| تمر سعودي طازج يُحصد من مزارع المدينة المنورة | 91.19 | 1.38 | 92.57 |
| شاي أسود هندي مستورد من أفضل مزارع دارجيلينج | 38.52 | 1.11 | 39.00 |

Actual throughput was ~5 msg/s against a 150 msg/s target. The embedding step was the
bottleneck — a lighter encoder was needed.

---

### Phase 3 — Harrier + XGBoost (production model)

**Model:** `microsoft/harrier-oss-v1-270m` + XGBoost  
**Training set:** 5,000 curated samples from the labeled pipeline output

#### Modality comparison (held-out test set, n = 866)

| Modality | Precision | Recall | F1 | ROC-AUC |
|---|---|---|---|---|
| Text embeddings only | 0.806 | 0.488 | 0.607 | 0.884 |
| Image embeddings only | 0.908 | 0.869 | 0.888 | 0.982 |

Text embeddings alone significantly under-recall. Many spam listings — particularly
medications and ambiguous activation codes — are not distinguishable from legitimate
listings by text alone. Image features close this gap.

#### Fusion strategy comparison (held-out test set, n = 866)

| Strategy | TP | FP | TN | FN | Precision | Recall | F1 | Accuracy |
|---|---|---|---|---|---|---|---|---|
| Noisy OR | 163 | 20 | 677 | 6 | 89.07% | 96.45% | 92.61% | 97.00% |
| Max | 159 | 16 | 681 | 10 | 90.86% | 94.08% | 92.44% | 97.00% |
| Weighted Average | 153 | 4 | 693 | 16 | 97.45% | 90.53% | 93.87% | 97.69% |

**Production strategy: Weighted Average** (0.3 × text + 0.7 × image)

Chosen because wrongly auto-removing a legitimate seller listing has a higher business
cost than missing a spam listing that gets caught on the next update cycle. Weighted
Average minimises false positives (4 FP) at the cost of slightly more missed spam
(16 FN), while still achieving the highest overall accuracy (97.69%).

**Training methodology:**
- 5-fold stratified cross-validation (weighted F1)
- Decision threshold sweep across 0.10–0.95 cutoffs
- `scale_pos_weight` calibration analysis using reliability curves and Brier score

---

## Error analysis

Validated on the first 100 rows of the held-out test set.

| Error type | Count | Pattern |
|---|---|---|
| False positives | 1 | Legitimate monthly subscription code flagged as spam |
| False negatives | ~60 | Medications (DICLOFENAC, OLMESARTAN) — classified as CLEAR |

The false negatives revealed a data labeling gap: pharmaceutical products use clinical
terminology that was not covered by the original spam lexicon used to generate training
candidates. These samples were relabeled and the model was retrained.

---

## Data labeling pipeline

| Stage | Input | Output |
|---|---|---|
| Raw Salla dataset | ~45,000 listings | — |
| Keyword extraction + weak labeling | — | ~7,000 high-confidence labeled samples |
| LLM review (GPT-4.1-mini, OpenAI Batch API) | — | Final ground-truth labels |
| Training subset for Harrier+XGB | — | 5,000 samples |

**Confidence routing during labeling:**

| Band | Action |
|---|---|
| > 0.85 | Auto-label |
| 0.60–0.85 | GPT-4.1-mini review |
| < 0.60 | Human review queue |
