# 002. Multimodal embeddings: text + image features fed to XGBoost

**Date:** May–June 2026

## Context

Arabic product listings carry both a text description and product images. Text alone
proved insufficient for many spam categories: medications use clinical terminology that
doesn't overlap with the spam training signal, and ambiguous digital goods rely on
visual packaging cues. The embedding model choice was also constrained — Salla's
production environment already runs specific models, so we had to use what was
available there.

## Alternatives considered

- **Text embeddings only** (Harrier text encoder)
- **Image embeddings only** (SigLIP `google/siglip2-base-patch16-224`)
- **Combined text + image** — two separate XGBoost models, probabilities fused

## Decision

**Combined text + image, Weighted Average fusion.** Two separate XGBoost models are
trained — one on image embeddings, one on text embeddings — and their output
probabilities are blended at inference:

```
final_proba = 0.3 × text_proba + 0.7 × image_proba
```

The higher weight on image reflects the large accuracy gap between the two modalities
on this dataset (see results). Both encoders are constrained to Salla's production
infrastructure.

## Decision matrix

| Criteria | Text only | Image only | Text + Image |
|---|---|---|---|
| Precision | 0.806 | 0.908 | 0.974 (Weighted Avg) |
| Recall | 0.488 | 0.869 | 0.905 |
| F1 | 0.607 | 0.888 | 0.939 |
| ROC-AUC | 0.884 | 0.982 | — |
| Inference complexity | Low | Medium | Higher — two encoders |
| Deployment fit | Good | Good | Both models in Salla infra |

## Consequences

Gained: all three fusion strategies exceed F1 0.92 — neither modality alone achieves
this. Precision reaches 0.974 with Weighted Average, meaning very few legitimate
listings are wrongly flagged.

Given up: two encoder models at inference increases memory and latency. Image
availability is assumed — listings without images need a fallback.
