# Better retrieval without giving up the context bound

**500 agents — same fresh holdout**

- V1 owner recall: **25.0%** (15/60)
- V2 owner recall: **86.7%** (52/60)
- V2 maximum roster context: **2.45 KB** (2,453 UTF-8 bytes)
- **Zero** violations of max 8 agents / 4,096 bytes

Offline; 14 fresh tasks × 3 sizes × 5 seeds = 210 conditions,
420 paired ranker executions. Recall excludes two tasks without a unique owner.

Speaker note: the ranker was frozen before holdout authoring and run once.
This is a small synthetic test, not proof of real-model routing. The 90% target
was not reached. Remaining failures: semantic paraphrases and topic changes.
Do not use the old 60% as the before number for this new holdout.

Evidence: `evals/results/retrieval-v2-heldout-once.json`, summary size=500.
