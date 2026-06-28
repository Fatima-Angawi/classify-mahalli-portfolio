# 006. Decision thresholds tuned to minimise HUMAN_REVIEW

**Date:** May 2026

## Context

Once the three-tier output was adopted, specific cutoff values had to be set. The
thresholds determine what fraction of listings go to each tier. In trust & safety,
the cost of a false negative (missed spam staying live) and a false positive (legitimate
listing auto-removed) are not equal, and moderator time has a real cost. The goal was
to push the classifier to commit to high-confidence outcomes wherever possible,
keeping the ambiguous middle tier narrow.

## Alternatives considered

- **Default 0.5 cut** — single threshold, no human review tier
- **Symmetric split at 0.5 / 0.75** — equal-width bands
- **Sweep-selected thresholds** — values chosen after running a precision-recall sweep
  and calibration analysis

## Decision

**Sweep-selected thresholds.** A threshold sweep was run across 0.10–0.95 cutoffs
to map the precision/recall trade-off. `scale_pos_weight` was tested at values 1, 2,
and the computed class ratio to verify probability calibration via reliability curves
and Brier score. The selected thresholds:

| Context | AUTO_REMOVE | HUMAN_REVIEW |
|---|---|---|
| Kafka streaming service | 0.65 | 0.5 |
| Mage.ai pipeline (Salla prod) | 0.7 | 0.4 |

The mage pipeline uses a higher AUTO_REMOVE bar and a lower HUMAN_REVIEW floor,
producing a wider CLEAR zone suited to the batch processing context.

## Decision matrix

| Criteria | Default 0.5 | Symmetric 0.5/0.75 | Sweep-selected |
|---|---|---|---|
| HUMAN_REVIEW volume | None (no middle tier) | Moderate | Small — narrow ambiguous band |
| False positive rate | High | Lower | Lower — high bar for auto-removal |
| Calibration requirement | Low | Moderate | High — probabilities must be trustworthy |

## Consequences

Gained: raising the AUTO_REMOVE bar above 0.5 reduces the risk of wrongly removing
legitimate seller listings. Calibration analysis confirmed the model's probabilities
are trustworthy enough to use as confidence signals rather than just hard predictions.

Given up: the full threshold sweep output is not committed to the repo — only the
final selected values and the methodology are documented.
