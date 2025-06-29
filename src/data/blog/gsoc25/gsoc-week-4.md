---
title: "GSoC Week 4: Enhancing OIE using ReAct prompting + Link Prediction"
author: "Aditya Venkatesh"
pubDatetime: 2025-06-28T16:00:00.000Z
slug: gsoc-week-4
featured: true
tags: ["GSoC", "Hindi", "LLM", "ReAct", "linkprediction", "analysis", "Week 4"]
description: "Week 4 of GSoC'25: Trying new LLMs, using the ReAct prompting strategy and analysing our graph for link prediction models. "
---


> The whole of science is nothing more than a refinement of everyday thinking. — Albert Einstein

## Introduction

In the 4th week of my journey to improve Hindi information extraction, I added on more models to experiments with. I pruned the useless strategies and models.  In addition to this I analysed the graph we formed using the DIEF from week 1. The goal is to analyse this graph, draw insights about its structure and come up with algorithms to predict missing links and complete this knowledge graph. 

---
## LLM powered relation extraction

As suggested by my mentors I tried a bunch of new models namely llama3.2 1B, llama3.2 3B, gemma3 4B - fp16, gemma3:12B 4 bit quantised. 
After running multiple experiments I narrowed down my models to gemma3:4b and gemma3:12b. It makes sense to only use gemma for extraction; in addition to being the best performing model in terms of its f1 score, it also is really really fast. I decided to keep just a few strategies in the end to compare how gemma performs across various prompts.
- Prompt Strategies: Few Shot (Baseline), Chain of Thought Evidence Based Reasoning, ReAct (Reason, then Act) Prompt
- Model Variants: Gemma3 4B, Gemma 3 12B


### Prompt Design Overview
The issue with our previous prompts, in simple terms, was that the LLM was trying to just generate the next token of sequence.
As per the [ReAct](https://react-lm.github.io/) paper, we can squeeze more performance out of LLMs by prompting them to reason over some information (context) and then act by responding in a particular format. These responses can be used as input to other tools, say an API, to collect more information and then continue the chain of reason-then-act till the LLM deems its job done. 
The LLM can have access to multiple tools but this depends on the context size our LLM can accept. In our trials I decided to limit the LLM to having access to one tool (extract_triplets) and one example of a correct extraction with tool usage. You can see the exact prompt along with the code [here](https://github.com/advenk/neural-extraction-framework/blob/gsoc25_main/GSoC25_H/ReAct/prompt_factory.py).

### Results
Just by guiding the LLM to respond in a specific format, we were able to reduce false positives from 107 to 86 for the same model. This gives us a boost in the f1 score, however this also results to a higher number of false negatives. Any small mismatch or loss in information while extracting from a complex sentence can lead to a false negative. 

Gemma3:12B (4 bit quantized) model was the ebst performing model with the following metrics:
```json
"overall_metrics": {
    "precision": 0.410958904109589,
    "recall": 0.2955665024630542,
    "f1_score": 0.34383954154727797,
    "total_true_positives": 60,
    "total_false_positives": 86,
    "total_false_negatives": 143
  }
```
---

## Link Prediction 

To setup and brainstorm the next steps for this projects, I decided to analyse the graph we have currently from our DIEF extraction for Hindi.
An initial brute force of constructing the graph gives us 2039012 triples. After removing the disambiguation relation and only keeping valid relations which have "ontology" in them we narrow down to 304395 triples. I call these the "core" triples. 
Constructing the knowledge graph with this we can see the sparness in this graph by modeling its degree (connectedness).

### Statistics

```json
count    304395.000000
mean          3.575361
std          84.226941
```

The given stats show that for every entity, on average it is connected to 3 other entities. The standard deviation of 84, however tells us that, some entities are much more connected than other hinting at the skewedness of the distribution. Therefore our graph is sparse and might not be the best as of now for link prediction.


### Relation analysis
Doing a simple analysis of the count of relations we can see that there are a bunch of relations which are redundant. 
- language (count:108087) occurs 108087 times. Technically it's semantic but its always referring to Hindi and hence is useless for link prediction tasks due the massive hub it creates around this one single langugage. 
- wikiPageRedirects (77046 count): This is not a semantic relationship. It's a structural artifact from Wikipedia that says "this page is a redirect to another". Keeping this will teach the model that many things are simply equivalent, which is not useful for predicting new facts. We can remove this safely.
- thumbnail (78098 count): This links an entity to its image URL. It has zero semantic value for predicting relationships like (Person, birthPlace, City). 


Relations like starring, occupation, nationality, director, subdivision, state, district, country, birthPlace, deathPlace, location, residence - these are the facts we want to model. Starring, producer, director, writer, musicBy, genre: form a strong sub-graph about media and films. Occupation, politician, party, almaMater: great biographical and political relations. All the others in the top 30 are generally good semantic relationships.


On the other hand, relations which have a very small frequency can also be pruned for the graph to be efficient and our subsequent models to learn better. A model can't learn from sprase relations which only show up a couple of times. Relations like bodyDiscovered (1), taoiseach (2), or mother (5) are too sparse. My plan is to set a threshold and remove any relation that appears fewer than 50 or 100 times. While we lose some specific facts, we can create a denser, more learnable graph for the model.

### Pruning Strategy
Based on this analysis, my strategy is to perform a two-step pruning process to create a smaller, denser, and more semantically coherent graph.
- Semantic Filtering: We will remove the relations which are semantically useless using a "blacklist".
- Frequency Pruning (K-Core Pruning): We will remove all relations that appear too infrequently and all entities that are not connected enough times.

When we split into train/val/test sets we should make sure that entities in our val/test also appear in train set.

## Reflections and Next Steps

This week, I was able to push the performance of open source LLMs to achieve a f1 of 34% for open information extraction tasks on hindi text. I decided to finalise the model for future tasks - gemma3:12B. 
In addition to IE for hindi, I also read up on link prediction methods and did an initial analysis of the Hindi graph, setting it up for the subsequent models to run. 

**Next up:**
- Plug in gemma3 in the indIE architecture, overriding the manual rules created by the original authors. Run as an experiment to see if this enhances performance. 
- See how we can generate synthetic data for finetuning gemma3 for our use case. Write a script for generating this with gemini 2.0 flash.
- Try link prediction using transE and convE models setting up a baseline.
- If possible try to use initial embeddings created from the wikipedia articles of each of the entities and see how the models perform with and without this. 