# Design Decisions

These are the significant architectural and modelling choices made during the project,
documented with the context, alternatives considered, and trade-offs accepted.

| # | Decision | Summary |
|---|---|---|
| [001](001-model-selection.md) | Model selection: Harrier+XGBoost | Three experiments driven by a throughput constraint; Harrier chosen to match Salla's production infrastructure |
| [002](002-multimodal-embeddings.md) | Multimodal: text + image → XGBoost | Image embeddings alone outperform text alone; Weighted Average fusion is production strategy |
| [003](003-kafka-streaming.md) | Kafka streaming over batch/REST | Native CDC fit; filter and detector decouple cleanly as independent services |
| [004](004-cdc-filter-service.md) | CDC filter as an isolated service | Embedding only runs on description-changed listings; independent scaling |
| [005](005-three-tier-decision.md) | Three-tier decision output | Middle tier routes genuinely ambiguous cases to humans instead of forcing a binary cut |
| [006](006-decision-thresholds.md) | Decision thresholds | Sweep + calibration targeting a narrow HUMAN_REVIEW band |
| [007](007-rule-engine-disabled.md) | Rule engine disabled | Salla chose fully automated ML over maintained keyword rules |
| [008](008-mageai-pipeline.md) | Mage.ai pipeline for production | Kafka built first; Salla requested mage.ai to fit their production ML platform |
