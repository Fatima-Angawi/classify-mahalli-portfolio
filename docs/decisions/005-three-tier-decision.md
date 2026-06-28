# 005. Three-tier decision output: AUTO_REMOVE / HUMAN_REVIEW / CLEAR

**Date:** May 2026

## Context

A binary classifier outputs a probability in [0, 1]. Converting that to an action
for a trust & safety workflow requires a threshold decision. A hard binary cut forces
every borderline listing through the same action. In a real marketplace, wrongly
auto-removing a legitimate listing has a different cost to missing a spam listing —
moderator capacity exists to handle genuinely ambiguous cases.

## Alternatives considered

- **Binary decision** (`REMOVE` / `CLEAR`) — single threshold, no middle tier
- **Three-tier** (`AUTO_REMOVE` / `HUMAN_REVIEW` / `CLEAR`) — two thresholds; high-
  confidence spam auto-actioned, ambiguous routed to humans
- **Raw probability passthrough** — downstream system owns the threshold logic

## Decision

**Three-tier decision.** Two thresholds applied to the classifier probability:

```
prob >= 0.65  →  AUTO_REMOVE   (HIGH_RISK)
prob >= 0.5   →  HUMAN_REVIEW  (MEDIUM_RISK)
prob <  0.5   →  CLEAR         (LOW_RISK)
```

## Decision matrix

| Criteria | Binary | Three-tier | Raw probability |
|---|---|---|---|
| Moderator workload | High — borderline cases auto-actioned | Controlled — only ambiguous zone | Pushed downstream |
| Seller impact | High FP risk at any threshold | Reduced — high bar for auto-removal | Downstream responsibility |
| Auditability | Low | High — tier + confidence in output | High |
| Adaptability | Threshold = code change | Threshold = config change | Downstream change |

## Consequences

Gained: auto-removal only triggers above a confidence bar that protects legitimate
sellers. The `HUMAN_REVIEW` tier creates a feedback loop — human decisions on
ambiguous cases feed back into retraining data. Confidence and tier in the output
event allow dashboards to prioritize the review queue.

Given up: the size of the `HUMAN_REVIEW` bucket depends on probability calibration.
A miscalibrated model routes most listings to human review and eliminates the
automation benefit — this is why calibration analysis was part of the training process.
