# 007. Rule-based policy engine disabled

**Date:** May 2026

## Context

During development I built a rule-based `PolicyEngine` alongside the ML classifier.
It used an Aho-Corasick automaton to match product text against curated keyword lists
(IPTV terms, restricted activation phrases, suspicious keyword pairs), short-circuiting
the ML classifier when a rule matched. After discussing the design with Salla's product
team, the decision was made not to rely on maintained rules in production.

## Alternatives considered

- **Rules + ML hybrid** — high-confidence keyword rules short-circuit the classifier;
  ambiguous cases go to the model
- **ML only** — classifier handles all cases; no maintained keyword lists

## Decision

**ML only.** The PolicyEngine is disabled. All decisions go through the XGBoost
classifier. The keyword rules and rule engine code are retained in the codebase but
inactive.

## Decision matrix

| Criteria | Rules + ML hybrid | ML only |
|---|---|---|
| Coverage of known-bad patterns | Excellent — explicit rules | Depends on training coverage |
| Maintenance burden | High — rule lists need updates as spam evolves | Low — model retrained periodically |
| Generalization | Poor for novel spam not in lexicon | Good — embedding captures semantics |
| Operational ownership | Requires a team maintaining rules | Self-contained in ML team |
| Decision consistency | Rules can conflict with model output | Single pathway |

## Consequences

Gained: no keyword list to maintain. As spam categories evolve, only training data and
the model need updating — the decision logic stays constant. A single classification
pathway simplifies auditing and debugging.

Given up: instant, zero-latency detection of known IPTV keywords and similar high-
confidence patterns. These now rely on the model having sufficient training coverage
of those categories.
