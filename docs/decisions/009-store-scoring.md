# 009. Store-level spam scoring as a product feature

**Date:** June 2026

## Context

Product-level classification treats each listing in isolation. But spam on Salla is
not always random — certain stores systematically list prohibited or restricted
products. A listing from a store with a 60% historical spam rate is a qualitatively
different signal to an isolated listing from an otherwise clean store. The question
was whether to add store-level context as a feature to the classifier.

## Alternatives considered

- **Product-level only** — classify each listing independently; no store history used
- **Store score as a feature** — compute per-store spam statistics, flag suspicious
  stores, enrich each product row with store context before classification
- **Store-level hard block** — auto-remove all listings from stores above a threshold
  without running the classifier per listing

## Decision

**Store score as a feature.** A store scoring function computes per-store statistics
and flags stores as suspicious when:

```
spam_rate >= 40%  AND  spam_count >= 10  AND  total_products >= 20
```

Each product row is enriched with a `store_context` field:

```
spam AND suspicious store      →  "spam_from_suspicious_store"
spam AND clean store           →  "spam_from_clean_store"
otherwise                      →  "normal"
```

Store scores are also written to a standalone table for downstream monitoring.

## Decision matrix

| Criteria | Product-level only | Store score as feature | Hard block |
|---|---|---|---|
| Catches systematic offenders | Poor | Good — store history raises signal | Good |
| False positive risk | Baseline | Low — still classifier-gated | High — no per-listing check |
| Explainability | High | High — context field in output | Low |
| Latency | Low | Low — scores precomputed | Very low |
| Maintenance | None | Scores need periodic refresh | Threshold tuning required |

## Consequences

**Gained:** Store context gives the classifier a prior — listings from high-rate
stores are treated as higher risk even when the text alone is ambiguous. The
standalone store scores table enables a monitoring view of which stores are trending
toward suspicious behavior, independent of per-listing decisions.

**Given up:** Store scores must be recomputed periodically as new labels accumulate.
A store that cleans up its behaviour may remain flagged until the next refresh.

## Parameters

| Parameter | Value | Rationale |
|---|---|---|
| `spam_rate_threshold` | 40% | Raised from default 30% to reduce false flags on small stores |
| `min_spam_count` | 10 | Requires meaningful volume of confirmed spam |
| `min_products` | 20 | Excludes stores with too few listings to be statistically reliable |
| `store_score` formula | `spam_rate × spam_count` | Penalises both high rate and high absolute count |
