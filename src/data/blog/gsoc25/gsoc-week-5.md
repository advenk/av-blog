---
title: "GSoC Week 5: Link Prediction and indIE Enhancement"
author: "Aditya Venkatesh"
pubDatetime: 2025-07-04T21:16:12.000Z
slug: gsoc-week-5
featured: true
tags: ["GSoC", "knowledgegraphcompletion", "linkprediction", "informationextraction", "Week 5"]
description: "Week 5 of GSoC'25: Implementing and benchmarking link prediction models for our existing Hindi knowledge graph"
---


> What we know is a drop, what we don't know is an ocean. — Isaac Newton

## Introduction

This is the 5th week of my journey with GSoC and enhancing the neural extraction pipeline for Hindi information extraction. Over the past week, I worked on expanding the existing link prediction notebook and finally implementing and benchmarking some link prediction models (namely TransE and ConvE) on our extracted hindi knowledge graph. I also tried out my idea of integrating a small language model like Gemma 3 into the existing indIE architecture. We come up with some surprising results!

---
## Link Prediction

### Pruning and Data Preperation
As discussed in the last week's post, I decided to prune the graph to have a semantically sensical final graph for the models to work with. By eliminating noisy relations and pruning the graph, we not only enhance model output metrics but also its training and execution efficiency.

```code
RELATION_BLACKLIST = [
    'http://dbpedia.org/ontology/wikiPageRedirects',
    'http://dbpedia.org/ontology/thumbnail',
    'http://dbpedia.org/ontology/language',
    'http://dbpedia.org/ontology/timeZone',
    'http://dbpedia.org/ontology/restingPlacePosition',
    'http://dbpedia.org/ontology/mainArticleForCategory'
]
```

The given relations were pruned. 

```
number of core triples before pruning: 544161
number of triples after removing blacklisted relations: 269108
```

We then prune the graph as per relation frequency and entity degree. These are the desired properties of the final graph:
```
MIN_RELATION_FREQUENCY = 50
MIN_ENTITY_DEGREE = 10
```

The corresponding triples produced are what I call "semantically filtered triples". Final number of high-quality triples after pruning: 10,183. 


### Model Training and Evaluation

I used the pykeen library which provides a simple interface for training and running knowledge graph models for various tasks, especially for link prediction. After prepping the data, model training is as simple as calling this function:

```code
pipeline_result_transe = pipeline(
    training=tf_train,
    validation=tf_validation,
    testing=tf_test,
    model='transE',
    model_kwargs=dict(embedding_dim=100),
    training_kwargs=dict(num_epochs=100, batch_size=128),
    optimizer_kwargs=dict(lr=0.01), 
    random_seed=42,
    device='mps'
)
```

I experimented with a few hyper parameter configurations and this was the one which worked best for training locally. 


#### Incorporating MURIL Embeddings
In essence all knowledge graph completion (KGC) models learn and modify embeddings of the entities to find similarities and new links for our graph. Our idea here is that we push the model to start with the initial embeddings for each of the entities which better represent the entities and captures its nuances based on the context of the language (which is Hindi here.)

[Muril embeddings](https://arxiv.org/abs/2103.10730#:~:text=LMs,MuRIL%20in%20handling%20transliterated%20data) are embeddings tailored for low resource languages like Hindi. Trained on data from Wikipedia, these serve as a good starting point to see how these embeddings can boost performance. I took the name of all the entities in our graph and ran it through the muril-base-cased tokenizer to get the (1 x 768) dimension vector which captures the nuances of the particular entity. 

One problem here is that each entity encoded with a 768-dimensional vector would be too much for my local system to handle. This is why we perform Principal Component Analysis (PCA) to reduce the dimensions to 200 which would be more efficient for training on our machine. Note: we sacrifice some information, i.e, we accept that the 200 dimension vector might and would lose some information as compared to its 768 dimension counterpart. 

Now we can train our model with the initial embeddings that we have derived using the muril based tokenizer:

```code
pipeline_result_transe_muril = pipeline(
    training=tf_train,
    validation=tf_validation,
    testing=tf_test,
    model='TransE',
    model_kwargs=dict(embedding_dim=200, entity_initializer=PretrainedInitializer(tensor=entity_embeddings_tensor)),
    training_kwargs=dict(num_epochs=100, batch_size=256),
    optimizer_kwargs=dict(lr=0.01), 
    random_seed=42,
    device='mps'
)
```

### Experiment Setup
Finally, we run and evaluate the performance of our two models (Transe, ConvE) with 2 different starting points (with and without explicit initial embeddings). The hypothesis here is the muril based embeddings **should ideally** give better results than its counterpart. 

### Results and Analysis

#### 1. Analysis of Latest Results with MIN_ENTITY_DEGREE = 10

Putting all four models' "realistic" tail prediction metrics side-by-side for a clear comparison:

| Model                       | `hits_at_1` | `hits_at_3` | `hits_at_5` | `hits_at_10` | `arithmetic_mean_rank` (Lower is better) | `inverse_harmonic_mean_rank` (Higher is better) |
| :-------------------------- | :---------- | :---------- | :---------- | :----------- | :--------------------------------------- | :---------------------------------------------- |
| **TransE (Default Init)**   | 0.0098      | 0.0353      | 0.0599      | 0.1129       | 147.49                                   | 0.0487                                          |
| **ConvE (Default Init)**    | 0.0049      | 0.0088      | 0.0118      | 0.0285       | 266.15                                   | 0.0185                                          |
| **TransE (MuRIL Init)**     | **0.0226**  | **0.0667**  | **0.0903**  | **0.1315**   | **137.47**                               | **0.0646**                                      |
| **ConvE (MuRIL Init)**      | 0.0147      | 0.0402      | 0.0579      | 0.0903       | 153.57                                   | 0.0446                                          |

**Key Observations:**

1.  **MuRIL Embeddings are Beneficial:** Both TransE and ConvE show an improvement when initialized with MuRIL embeddings (after PCA reduction).
    *   **TransE (MuRIL)** is the **best performing model** across all `Hits@k` metrics, and has the lowest `arithmetic_mean_rank` and highest `inverse_harmonic_mean_rank`. This is a clear win for using MuRIL embeddings with TransE.
    *   **ConvE (MuRIL)** shows a very significant improvement over **ConvE (Default)** across all metrics. For instance, its Hits@10 jumped from 0.0285 to 0.0903. This indicates that MuRIL embeddings provide a much better starting point for ConvE, helping it learn more effectively.

2.  **TransE still outperforms ConvE:** Even with MuRIL initializations, TransE (MuRIL) still slightly outperforms ConvE (MuRIL) on this dataset and hyperparameter set. This is still a bit counter-intuitive given ConvE's theoretical capacity, reinforcing the idea that ConvE is more sensitive to its setup. Tuning the hyperparameters here would be a promising approach to extract more performance. 

**Potential areas for refinement:**

*   **Richer Entity Representations:** Instead of just the entity name, we could try to embed a short textual description of the entity (first paragraph from wikipedia) using MuRIL to get a more comprehensive semantic representation. 
*   **Fine-tuning MuRIL:** In very advanced setups, the MuRIL model itself could be fine-tuned *during* the KGE training process, allowing the text embeddings to adapt further to the knowledge graph structure. we can explore ways to do this with built-in PyKEEN methods. 

### 3. What we can try next:


**Hyperparameter Tuning:**

*   **Systematic Tuning:** For a serious hyperparameter search, consider using tools like:
    *   **Grid Search:** Trying all combinations of a predefined set of hyperparameters (can be computationally expensive).
    *   **Random Search:** Randomly sampling hyperparameters from a distribution (often more efficient than grid search for complex models).
    *   **Bayesian Optimization (e.g., Optuna, Ray Tune):** More advanced methods that intelligently explore the hyperparameter space.

**Other KGE Models:**

*   Exploring other models might be beneficial like:
    *   **DistMult** 
    *   **RotatE** 
    *   **ComplEx**
    *   **GNNs** 

---

## Integrating a LM into the indIE architecture

### Core Idea
We have seen that [indIE](https://aclanthology.org/2023.findings-ijcnlp.28.pdf) set the state-of-the-art for Hindi information extraction back in 2023 with the following metrics on the BenchIE dataset:
```
Precision   0.49
Recall      0.53
F1 Score    0.51
```

The idea here is simple. IndIE relies on a three-stage pipeline to produce its final output:
Sentence (input) -> Chunking (P1) -> Creation of MDT (P2) -> Handwritten Rules (P3) -> Triplets (Output)

Let's look at what each of these phases does at a high level:
1. **Chunking**: This is the process of breaking down the input sentence into meaningful multi-word units. Each chunk represents:
- Noun phrases (entities, objects)
- Verb phrases (actions, states)  
- Prepositional phrases (relationships, locations, times)
- Other syntactic units

For example:
Sentence: कार्यरूप जगत को देखकर ही शक्तिरूपी माया की सििद्ध होती है .
Chunks: ['कार्यरूप जगत को', 'देखकर ही', 'शक्तिरूपी माया की', 'सििद्ध होती है', '.']

2. **Merged Dependency Tree**: The MDT shows how chunks relate to each other grammatically, helping identify subjects, objects, and modifiers. For example for the same above sentence, we would derive the following MDT: 
```
Root Phrase: "सििद्ध होती है" (main predicate/action)
Dependency Relations:
  - कार्यरूप जगत को->obj
  - देखकर ही->advcl
  - शक्तिरूपी माया की->nmod
  - सििद्ध होती है->root
  - .->0
```

3. **Handwritten Rules for Triplet Extraction**: For the final output, indIE uses over 100 handwritten rules to derive the final output. This is the brittle component that we want to enhance. 

The idea here is to try replacing the component of hand written rules. Instead of relying on these, we pass the chunks and MDT to a small LM like Gemma 3. We explain to Gemma 3 deeply what a dependency tree is and how it works. Once the model is provided with all this information, it should ideally be able to generate better triplets than before. 


### Setup and Results
I ran 2 experiments: 
1. Passing the entire MDT and chunks and relying on LLM to perform the entire extraction. The results of such a run are:
```
=== FINAL METRICS Gemma 3 - 12b===
Total TP (True Positives): 145
Total FP (False Positives): 380
Total FN (False Negatives): 113
Recall 0.562015503875969
Precision 0.2761904761904762
F-score 0.3703703703703704
```

2. After the indIE rule application, we ask gemma to filter. I use and evaluate two different prompts.  

Prompt 1: Less conservative, gemma filters more freely.
```
=== FINAL METRICS Filtering - After rule application, ask gemma to filter ===
Total TP (True Positives): 107
Total FP (False Positives): 104
Total FN (False Negatives): 136
Recall 0.4403292181069959
Precision 0.5071090047393365
F-score 0.4713656387665198
```

Prompt 2: Push gemma to be more conservative. 
```
=== FINAL METRICS Filtering - ask model to be conservative ===
Total TP (True Positives): 117
Total FP (False Positives): 140
Total FN (False Negatives): 126
Recall 0.48148148148148145
Precision 0.45525291828793774
F-score 0.46799999999999997
==============================================
```

We can see the precision-recall tradeoff at play here, although it's minor.



## Conclusion and Next Steps

### Link Prediction
As discussed with the mentors, we see that our Hindi knowledge graph is not mature enough for link prediction models to perform very well. The benchmark we have set with our prototype is a baseline for future implementations. Going forward, we will not work on link prediction, at least in the near future, but divert focus to other avenues. 

### Information Extraction
We see that the filtering of indIE triples with a LM like gemma can help improve performance. Also using the LM for just enhancement also pushed the recall much higher, though it lags in precision. Our aim is to try out a few different things to increase recall while also keeping precision so as to achieve state-of-the-art performance. Over the next couple of weeks this is what I'll be focusing on: 

1. Improve the prompt for extracting new triplets by incorporating the following suggestions:
- Directly mention it's a dependency tree and pass it as triples as given below:
```
Sentence: "Barack Obama was born in Hawaii."
Dependency Tree:
- (was, Obama, nsubjpass)
- (was, born, auxpass)
- (born, was, root)
- (born, in, prep)
- (in, Hawaii, pobj)
```

- Add instructions to keep chunks intact. 

2. Try the following and see performance:
Rule extraction + Enhancement + Filtering. Theoretically, this should give us better performance than our current setup. 

3. Manual analysis of where the LLM is going wrong. Aim here is to compare and analyse where exactly the LM is going wrong. Is it missing some properties? What is making the FNs and FPs occur? What is the LM not good at? 
This will help us map out how to proceed further and extract the most performance possible. 

4. Get synthetic data for hindi relation extraction by running the script with Dr. Ronak.

5. Setup a high level plan of finetuning + evaluation using the produced synthetic data. 

---







