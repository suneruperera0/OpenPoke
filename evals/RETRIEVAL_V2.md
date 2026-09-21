# Retrieval v2: development checkpoint

Status: final development pass completed; scorer frozen before fresh holdout authoring.
No model calls, external dependencies, embeddings, or paid APIs are used.
The historical v1 held-out report (including 60% recall at 500 agents) is unchanged.

## Design

`directory.py` retains the directory format, references, exact-reference priority,
recent-delegation priority, search API, pagination and renderer. The initial roster
block is still capped at eight records and 4,096 UTF-8 bytes, including its envelope.
This is a roster-block cap, not a bound on the entire conversation prompt.

The scorer is a field-aware BM25 variant, not a claim of canonical BM25F:

- Names have weight 1; responsibility/history has weight 2.
- Purpose and recent instructions are combined by maximum per-term frequency,
  avoiding repeated evidence from duplicate metadata.
- Term frequency is field-length normalized (b=0.5), with the length penalty capped
  at 1.5 to limit dilution from long histories. The strongest normalized
  field supplies each term's evidence, followed by BM25 saturation (k1=1.2).
- Document frequency is measured once across each agent's combined fields.
- Simple plural normalization affects lexical scoring only, never identities.
- Latest-request terms have weight 1; previous-user-context terms weight 0.5.
- Exact-reference and recent-delegation priority remain unchanged.

The final development pass adds a small, explicit general lexical alias map
(e.g. vendor/supplier, purchase/procurement) and strips only comma-delimited
", not ..." clauses from positive query evidence. These are limited heuristics:
renewal/extension and similar aliases are not equivalent in every domain, and other
negation forms are not handled. Names and references are never normalized for identity.
The scorer remains self-contained and standard-library-only.

## Reproduce from repository root

Use the project virtual environment with `server/requirements.txt` installed for
integration tests. The evaluation itself can also run with standard-library Python.

```bash
.venv/bin/python -m unittest discover -s evals -p 'test_*.py' -v
.venv/bin/python -m evals.retrieval_v2_eval --output evals/results/my-retrieval-v2-dev.json
```

Choose a fresh output name; the evaluator refuses to overwrite evidence.
V1 is loaded directly from commit `fd7b71d3713237a0f0f60ef5871c1589b9ce01e0`.
V2 uses the current directory source. Source/fixture hashes are recorded.

## New development fixtures and counting

14 authored development tasks: 12 unique-owner tasks, one new responsibility,
and one ambiguous request. They cover explicit identities, opaque names, plurals,
contextual follow-ups, overlapping work, exclusions, paraphrases, old owners,
longer histories, topic changes and shared contacts.

Each task has five seeds and nested 10/100/500-agent rosters. A condition supplies
exactly the same roster, history, user request and recent-owner state to v1/v2;
the saved fixture hash allows checking this pairing. Metadata comes through the
production legacy-history migration. Expected-owner labels never enter scoring.

- **210 conditions** = 14 tasks × 3 sizes × 5 seeds.
- **420 ranker executions** = 210 conditions × 2 rankers, per development run.
- Recall denominator at each size: **60** = 12 unique-owner tasks × 5 seeds.
- New/ambiguous cases exercise bounds only, not model creation/clarification quality.

These are intentionally dense synthetic hard negatives: many agents share project,
contact or responsibility, some near-duplicate responsibilities recur. They are not
representative usage estimates. Seeds are correlated repetitions of 14 tasks, not
210 independent language-understanding examples. The recent-owner fixture is a
synthetic unrelated pin; it does not validate real-world recency behavior broadly.

## Development results

| Agents | V1 shortlist recall | V2 shortlist recall | V2 max roster bytes |
| ---: | ---: | ---: | ---: |
| 10 | 100% | 100% | 2,073 |
| 100 | 16.7% | 83.3% | 2,507 |
| 500 | 16.7% | 78.3% | 2,675 |

Shortlist recall is measured after rendering/byte trimming. First-search recall
happened to match shortlist recall here. Every measured block remained within both
bounds. The 500-agent v2 score is 47/60 successes, versus 10/60 for v1.

Two development runs are retained:

1. `retrieval-v2-dev-1.json`: field scores were added independently; both rankers
   scored 16.7% at 500. Inspection showed descriptive wrong-project agents received
   duplicated lexical credit from names and history.
2. `retrieval-v2-dev-2.json`: strongest-field pooling before saturation improved
   v2 to 78.3%. Remaining 500-agent misses: paraphrase 5/5, long history 5/5,
   excluded topic 3/5. This was the intermediate development implementation.

The first run's source hash differs from current source; its output is historical
iteration evidence, not the final scorer's result. Both runs used identical fixtures.
There was one scoring revision; no attempt was made to force the 90% aspiration.

## Final development pass and freeze

Exactly one final pass addressed paraphrases, long histories and comma-delimited
exclusions. `retrieval-v2-dev-final.json` records 100% v2 recall at all three sizes,
with max bytes 2,057 / 2,507 / 2,675. V1 remained 100% / 16.7% / 16.7%.
This is development evidence, not a held-out generalization claim.

`retrieval_v2_freeze.json` records scorer version, code/test/evaluator/development
fixture hashes and freeze time. After that, `retrieval_v2_holdout.py` was authored
with 14 new tasks, new identities, duties, phrasing and distractor templates.
`retrieval_v2_holdout_freeze.json` records its hash before evaluation.
Neither scorer nor holdout was edited after observing holdout results.

The fresh holdout was run once, with both rankers on identical fixtures:

```bash
.venv/bin/python -m evals.retrieval_v2_eval --split heldout --output evals/results/my-fresh-holdout.json
```

This command reproduces evidence; its output must never be used to retune on this
holdout. Source hashes can be checked against the freeze manifests first.

Holdout counting: 14 authored tasks × 3 sizes × 5 seeds = 210 conditions,
420 ranker executions. At each size there are 60 unique-owner recall observations
(12 tasks × 5 seeds). New/ambiguous cases check bounds, not model decisions.
Synthetic repeated distractors and correlated seeds limit generalizability.
The fresh holdout includes one semantic paraphrase beyond the alias vocabulary.
This is a single-author synthetic evaluation, not independently authored validation.

## Fresh holdout results (single run)

| Agents | V1 recall | V2 recall | V2 max roster bytes | Bound violations (both) |
| ---: | ---: | ---: | ---: | ---: |
| 10 | 100% | 100% | 2,371 | 0 |
| 100 | 25% | 90% | 2,380 | 0 |
| 500 | 25% | 86.7% | 2,453 | 0 |

At 500: v1 15/60, v2 52/60. V2 missed 5/5 semantic-gap and 3/5 topic-switch
conditions. It improved by 61.7 percentage points on these same fixtures but
missed the 90% target at 500 and the 95% target at 100. No subsequent tuning.
First-search recall equals shortlist recall in this run. Bytes measure only the
rendered roster block; the rest of the model prompt is outside this budget.

Source: `results/retrieval-v2-heldout-once.json`, `summary` and `runs`.
The original held-out 60% remains separate historical evidence.
23 routing tests passed before scorer freeze. No live calls were made.
