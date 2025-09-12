---
title: "GSoC Weeks 11-12: Predicate Linking Integration and Final Week Touches"
author: "Aditya Venkatesh"
pubDatetime: 2025-08-31T05:00:00.000Z
slug: gsoc-week-11
featured: true
tags: ["GSoC", "predicatelinking", "knowledgegraphs", "Week 10"]
description: "Weeks 11-12 of GSoC'25: Integrating Predicate Linking and Closing the Project"
---

## Introduction 
The predicate linking module implemented last week was demoed to the mentors and I got some valuable feedback which I worked on this week. I also implemented a LLM-as-a-judge script to use gpt-oss-120b model to score our synthetically generated data points. Finally, the repo code was cleaned up and made ready for final submission along with thorough documentation. 

---

## Predicate Linking

1. The lexical score currently was generated but without translation. For this, I implemented it so that the candiate was first translated to english and then compared lexically. This gives us a more meaningful comparison score.
2. Implemented wikidata to dbpedia mapping so that we get the DBpedia property link as output. 
3. Integrated the predicate linking module into the CLI and streamlit demo.
4. Added `link_predicate_batch` to the module to process candidates efficiently in batches.

---
## LLM-as-a-judge Scoring Script

In the previous weeks, we implemented a synthetic data generation pipeline for generating data to finetune a small LM locally. The problem with this data was that there was inconsistency in quality. I wrote a script to use the same gpt-oss model used for generation to also act as a judge and score each of the data points. 

This opens the door for eventual filtering based on the score of the data points. 

--- 

## Final Week 
- Final work submission URL - https://gist.github.com/advenk/8ce1bea298ca5c13829c8737bc21cc93 
- Cleaned and raised the final [PR for NEF](https://github.com/dbpedia/neural-extraction-framework/pull/20)
- Collected reference material for drafting a position paper for the work done in GSoC for the Hindi chapter over the last two years. 

---
## What’s Next

- Fix the "type" relation handling in predicate linking (generation + scoring).
- Add marginalization and reranking to EL (subjects/objects) beyond mGENRE top-1.
- Tighten extraction→linking interface so EL can be evaluated with gold/clean triples.
- Finetune the gemma3-4b model using the filtered data with data point scores above 8.
- Flesh out the workshop position paper which showcases our work on the Hindi chapter over the last two years. 