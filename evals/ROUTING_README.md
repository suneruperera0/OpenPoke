# Bounded agent discovery: evaluation report

## Problem and implementation

Pinned baseline: `5b5f635935a64ab37884c025d70abb0ed731c094`.

The baseline inserts every persistent agent name into every interaction prompt.
Its roster has no purpose metadata, and dispatching an unknown name immediately
creates another agent. This project replaces that path with:

- versioned directory records derived from real instructions/history;
- stable opaque references that preserve arbitrarily long exact identities;
- an eight-candidate, 4,096-byte initial shortlist;
- lexical-IDF ranking over names, entities, purpose and recent instructions;
- paginated `search_agents` discovery over the whole directory;
- existing-owner-only dispatch and deliberate `create_agent` creation;
- routing traces for candidates, bytes, searches, dispatch/creation and iteration
  exhaustion.

No embeddings, model-generated metadata, batching changes, or UI changes are in
scope. Similar names are never merged. Legacy name lists migrate once with a
`.legacy.bak` copy, and existing agent names/log identities remain unchanged.

## Reproduce without network or credits

From the repository root:

```sh
.venv/bin/python -m unittest discover -s evals -p 'test_*.py' -v
.venv/bin/python -m evals.routing_eval --split dev --output evals/results/my-routing-dev.json
.venv/bin/python -m evals.routing_eval --split heldout --output evals/results/my-routing-heldout.json
```

The evaluator refuses to overwrite evidence files. Use fresh filenames. It loads
the baseline prompt renderer directly from the pinned Git commit and runs the
treatment directory in temporary storage. It never imports application startup,
reads `.env`, calls a model, or touches the real roster.

## Fixtures and metrics

There are 14 authored tasks: seven development tasks and seven held-out tasks.
Each split covers exact reference, contextual follow-up, paraphrase, overlapping
responsibilities, old owner, new responsibility, and ambiguity. Held-out tasks use
different people, projects, naming styles, overlaps and historical facts.

Each authored task runs against 10, 100 and 500-agent rosters with five shuffled
distractor seeds: 105 executions per split, 210 total. Synthetic sizes are stress
levels, not claims about typical product usage.

The retrieval query is the original user request plus the same bounded recent-user
context used in production. No ideal evaluator query is supplied. New and ambiguous
tasks are excluded from owner-recall denominators because they have no unique
correct existing owner.

`name_only` uses the treatment ranker with names alone. `enriched` adds metadata.
This isolates metadata's retrieval contribution. The baseline measurement is its
full roster byte size and availability of every owner; it is not a baseline model
routing score because no paid model evaluation was run.

## Results

Development results were used to tune and freeze `lexical-idf-v2`. Held-out
results were then run once without retuning.

| Split | Agents | Name-only shortlist / search recall | Enriched shortlist / search recall | Max treatment roster bytes | Mean baseline bytes |
| --- | ---: | ---: | ---: | ---: | ---: |
| Development | 10 | 84% / 40% | 100% / 100% | 2,570 | 503 |
| Development | 100 | 40% / 40% | 100% / 100% | 2,801 | 4,907 |
| Development | 500 | 40% / 40% | 100% / 100% | 2,909 | 24,966 |
| Held-out | 10 | 80% / 20% | 100% / 100% | 2,684 | 501 |
| Held-out | 100 | 20% / 20% | 92% / 92% | 2,938 | 4,905 |
| Held-out | 500 | 20% / 20% | 60% / 60% | 3,083 | 24,964 |

The bound held in every run. At 500 agents the shortlist was about 88% smaller
than the baseline by bytes in the worst reported treatment case. Metadata greatly
improved retrieval, but lexical retrieval missed 40% of held-out owners at 500.
That failure is part of the result: v1 bounds context but does not preserve adequate
owner recall for every dense, overlapping roster.

The first-search and shortlist recall are identical in this benchmark because both
use the original request/context and the same eight-result ranker. `search_agents`
still matters operationally: the model can reformulate a weak query or paginate an
empty listing, but that behavior needs live or scripted-agent evaluation.

## Scripted integration evidence

The offline tests demonstrate:

- shortlist/search results can dispatch by stable reference;
- unknown names return plausible owners and do not create anything;
- explicit creation atomically creates once and reuses an exact name thereafter;
- similar names stay separate and conflicting name/reference inputs fail;
- a normal one-search/one-dispatch path fits within eight iterations;
- repeated searching is recorded as iteration exhaustion;
- legacy migration, malformed storage, byte limits, long names, cursor staleness,
  pagination reachability, concurrent creation, recent owner updates and clear
  behavior work as specified.

These are scripted integration checks, not evidence that Claude understands the
new tools. The demo fixtures are frozen in `routing_fixtures.py`. In the development
demo, both explicit and contextual follow-up owners are found in the initial
shortlist; no recovery miss was manufactured. Search recovery is exercised through
the tool integration test.

## Limits and next decision

This offline evaluation establishes bounded context, retrieval quality, and
scripted tool-flow correctness. It does not establish real-model routing,
clarification, creation behavior, end-to-end task quality, latency, or cost.
Those measurements belong to the separate live-model evaluation.

At 500 agents, enriched lexical recall falls to 60% on the untouched held-out
set. Do not tune further on that set. Any retrieval improvement should be chosen
using new development fixtures and measured on a newly frozen holdout.

The tracked evidence files are:

- `evals/results/routing-dev-v3.json`
- `evals/results/routing-heldout-v1.json`
