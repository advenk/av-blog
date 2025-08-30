---
title: "GSoC Weeks 10-12: Predicate Linking, Data Curation, and Pipeline Cleanup"
author: "Aditya Venkatesh"
pubDatetime: 2025-08-28T05:00:00.000Z
slug: gsoc-week-10-12
featured: true
tags: ["GSoC", "entitylinking", "evaluation", "dataprep", "Week 10-12"]
description: "Weeks 10–12 of GSoC'25: implementing predicate linking for Hindi DBpedia, curating synthetic data with LLM-as-a-judge, debugging EL, and final documentation."
---

> Growth is a series of small, correct edits—one link, one script, one eval at a time.

## Introduction

These two weeks were about tightening the loop between extraction quality, entity/predicate linking, and data curation. I revisited our synthetic data pipeline and implemented multi-signal predicate linking for Hindi DBpedia. I then finally cleaned up all the code,pushed the final changes and created my final work submission document. 

---

## Synthetic Data: Simpler Prompts, Stricter Judging

- Created a new synthetic data generation script without "thought process" (7 Aug). The goal was to simplify outputs and reduce drift.
- Evaluation results:
  - **Open-source model**: Fluent sentences but poor triplet fidelity (logical reversals, span drift, occasional grammar issues).
  - **o4-mini**: Only ~20% high-quality points; similar logical/span issues and one outright ungrammatical sample (likely Azure variant behavior).

Conclusion: Without stronger constraints and post-filters, extraction quality remains the bottleneck and contaminates downstream linking.

---

## Entity Linking: Reframing Evaluation and Dependencies

- Sampled 100 sentences from random hiwiki articles to probe EL behavior end-to-end.
- The current pipeline’s EL quality is tightly coupled with triplet correctness; when extractions are wrong, EL fails by construction.
- Key gaps identified:
  - Predicates were not linked (surface text only).
  - Subjects/objects rely on mGENRE top-1 with no marginalization or reranking.

Action: Decouple evaluation—first obtain high-confidence triplets, then measure EL independently.

---

## Predicate Linking: Multi-Signal Candidate Ranking

I implemented predicate linking with a composite scoring function that blends:

- **Graph priors** (schema/domain-range/type constraints)
- **Lexical similarity** (post translation to English)
- **Embedding similarity**
- **Type compatibility**

Ontology source: [DBpedia ontology snapshot](https://akswnc7.informatik.uni-leipzig.de/dstreitmatter/archivo/dbpedia.org/ontology--DEV/2025.08.12-001007/ontology--DEV_type=parsed.ttl).

Blacklisted properties (structural/noisy):

```text
http://dbpedia.org/ontology/wikiPageWikiLink
http://dbpedia.org/ontology/wikiPageRedirects
http://dbpedia.org/ontology/wikiPageDisambiguates
http://dbpedia.org/ontology/wikiPageExternalLink
http://dbpedia.org/ontology/wikiPageDisambiguation
http://dbpedia.org/ontology/wikiPageRevision
http://dbpedia.org/ontology/wikiPageSource
http://dbpedia.org/ontology/wikiPageRevisionID
http://dbpedia.org/ontology/hraState
http://dbpedia.org/ontology/logo
```


- Added `link_predicate_batch` to process candidates efficiently in batches.
- Normalized Wikidata→DBpedia resource mapping to stabilize downstream joins.

---

## Lexical Mapping and Candidate Generation

- Implemented lexical candidate generation after translating predicate mentions to English.
- Built a translation-aware lexical index for predicate strings, labels, and aliases.
- Verified Wikidata↔DBpedia class/property normalization for consistent scoring spaces.

---

## LLM-as-a-Judge: From 20k → 10k

- Wrote a lightweight judge script using (LLM-as-a-judge) to score and filter finetuning datapoints.
- Practical outcome: Got a final output in which there is a `score` for each data point on a scale from 1-10. 

---

## Bugs, Gaps, and Fixes

- Predicate linking currently only handles relational linking but not type linking. Type linking would require us to identify if a triplet is of the form (subject, "है", object) and then link it as the subject -> type -> object. For other triplets (relational), this is handled. The major difference is the "object" in a type linking is a class in the ontology whereas in regular relational linking this is a property. 
- EL remains brittle when upstream triplets are noisy; next iteration should add marginalization over mGENRE candidates and a reranker.

---

## Writing, References, and Cleanup

- Final work submission URL - https://gist.github.com/advenk/8ce1bea298ca5c13829c8737bc21cc93 
- Cleaned and raised the final [PR for NEF](https://github.com/dbpedia/neural-extraction-framework/pull/20)
- Collected reference material (thanks, Tom) for drafting a position paper for the work done in GSoC for the Hindi chapter over the last two years. 
- Opened a final PR to clean, rewrite, and document the pipeline (linking, data filters, and evaluation harness).

---

## What’s Next

- Fix the "type" relation handling in predicate linking (generation + scoring).
- Add marginalization and reranking to EL (subjects/objects) beyond mGENRE top-1.
- Tighten extraction→linking interface so EL can be evaluated with gold/clean triples.
- Iterate on judge criteria and sampling to further improve finetuning data.
- Flesh out the workshop paper: methods, ablations, and error analysis.


