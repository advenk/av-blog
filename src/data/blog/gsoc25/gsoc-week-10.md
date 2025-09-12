---
title: "GSoC Week 10: Predicate Linking"
author: "Aditya Venkatesh"
pubDatetime: 2025-08-13T05:00:00.000Z
slug: gsoc-week-10
featured: true
tags: ["GSoC", "predicatelinking", "knowledgegraphs", "Week 10"]
description: "Week 10 of GSoC'25: implementing predicate linking for Hindi DBpedia"
---

## Introduction

This week I worked on the task of predicate linking. Predicate linking is the task of linking the relation in (subject,relation,object) triplets to the overlying ontology.

---

## Predicate Linking
Upon analysing last years work I realised that our pipeline incorporates entity linking but not predicate linking. The relations in our triplets are kept as surface form text itself. The task now is to link the surface form texts to the DBpedia ontology.

### Methodology

The first step is to obtain the DBpedia ontology. This is downloaded from [here](https://akswnc7.informatik.uni-leipzig.de/dstreitmatter/archivo/dbpedia.org/ontology--DEV/2025.08.12-001007/ontology--DEV_type=parsed.ttl).

When multiple candidate properties are possible, a scoring mechanism is required to rank them and choose the best one. We combine several pieces of evidences to get a final score for each candidate:


1. **Graph Based Scoring**: Are these entities actually linked by this cadidate property in the DBpedia exisiting KG?
2. **Semantic Score**: Calculated by comparing the embeddings of the predicates in the semantic space, this answers the question: How similar is the property's description to the input surface predicate?
3. **Lexical Score**: How similar are the property's labels to the input surface predicate, purely base on string matching?
4. **Type Compatibility Score**: Do the domain and range of the property align with the types of the subject and object entities? 

We take the weighted sum of these scores to get the final score of the candidates and ranked them in descending order. 

There is also a blacklisted properties list which filters our candidates which are too generic or administrative like "wikiPageWikiLink". 

### Examples
```
TEST CASE 1/6

RESULT
Input: 'Sachin_Tendulkar' | 'जन्म स्थान' | 'Mumbai'
Top Property: http://dbpedia.org/ontology/birthPlace
Best Label (en): birth place
Score: 0.385
Direction: none
Evidence: {"graph": 0.0, "emb": 0.8877218961715698, "lex": 0.09523809523809523, "type": 1.0}
--------------------

TEST CASE 2/6

RESULT
Input: 'Microsoft' | 'द्वारा स्थापित' | 'Bill_Gates'
Top Property: http://dbpedia.org/ontology/knownFor
Best Label (en): known for
Score: 0.417
Direction: O->S
Evidence: {"graph": 1.0, "emb": 0.0, "lex": 0.08695652173913043, "type": 0.0}
--------------------

TEST CASE 3/6

RESULT
Input: 'Satyajit_Ray' | 'ने निर्देशित किया' | 'Pather_Panchali'
Top Property: http://dbpedia.org/ontology/director
Best Label (en): film director
Score: 0.413
Direction: O->S
Evidence: {"graph": 1.0, "emb": 0.0, "lex": 0.06666666666666667, "type": 0.0}
```

## Limitations and Future Work
This module gives us a good baseline for linking predicates to the dbpedia onotoloy. However there is one major limitation as of now. The linking currently only handles relational linking but not type linking. Type linking would require us to identify if a triplet is of the form (subject, "है", object) and then link it as the subject -> type -> object. For other triplets (relational), this is handled. The major difference is the "object" in a type linking is a class in the ontology whereas in regular relational linking this is a property. 

Future work would focus on addressing this major limitation. In the coming weeks, I will integrate this module in the existing pipeline both in the streamlit demo as well as the command line workflow.

