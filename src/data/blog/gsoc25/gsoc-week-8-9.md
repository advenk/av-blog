---
title: "GSoC Week 8-9: Paper Review, SPARQL Endpoint Demonstration, and Synthetic Data Generation"
author: "Aditya Venkatesh"
pubDatetime: 2025-08-06T05:00:00.000Z
slug: gsoc-week-8-9
featured: true
tags: ["GSoC", "paperreview", "sparql", "syntheticdata", "multilingual", "Week 8-9"]
description: "Weeks 8-9 of GSoC'25: Reviewing multilingual SPARQL paper, demonstrating Hindi DBpedia endpoint capabilities, and generating synthetic data for finetuning"
---

## Introduction

Over the past two weeks, I did a diverse set of activities that span from academic paper review and synthetic data generation. Week 8 focused on reviewing a significant paper on multilingual question answering over knowledge graphs, while Week 9 was dedicated to demonstrating our Hindi DBpedia SPARQL endpoint capabilities and generating synthetic data for model finetuning.

---

## Paper Review: Multilingual Text-to-SPARQL

### Paper Overview
I conducted a comprehensive review of the paper "Text-to-SPARQL Goes Beyond English: Multilingual Question Answering Over Knowledge Graphs through Human-Inspired Reasoning" during our weekly meeting. This paper presents a novel approach to multilingual question answering over knowledge graphs using a human-inspired reasoning framework.

### Strengths Analysis

**Comprehensive Experimental Setup**
The paper's experimental methodology was particularly impressive, featuring:
- Evaluation across 10 languages including low-resource ones
- Use of both commercial (OpenAI GPT-3.5 and GPT-4) and open-source models (Qwen and Llama)
- Detailed cost analysis for practical deployment considerations

**Valuable Ablation Study**
The ablation study was especially noteworthy as it systematically broke down the contributions of different components:
- Experience pool effectiveness
- Feedback step improvements
- Quantified gains from each architectural component

This granular analysis provides clear insights into which parts of the system contribute most to performance improvements.

### Areas for Improvement

**Lack of Error Analysis**
The most significant gap identified was the absence of detailed error analysis. While the paper shows where the agent performs well, it doesn't explain:
- Specific failure conditions and patterns
- When the agent creates flawed execution plans
- Whether NEL (Named Entity Linking) tool failures contribute to errors
- Performance degradation with increasing query complexity (constraints, counting, ordering)

This analysis would be invaluable for determining future work directions and improvement areas.

**Limited Feedback Loop**
The feedback mechanism was identified as overly simplistic:
- Only single-pass correction implemented
- Authors mention avoiding infinite loops as justification
- Potential for multi-step self-correction or debate-based refinement
- Could explore iterative refinement strategies

**Inconsistent Language Prompting**
The paper used a mix of native speaker-written prompts and machine-translated ones without discussing:
- How this inconsistency affected performance
- Comparative analysis of native vs. translated prompts
- Impact on low-resource language performance

---

## Hindi DBpedia SPARQL Endpoint Demonstration

### Motivation and Benefits

The deployment of a dedicated Hindi DBpedia chapter represents a significant milestone in making structured knowledge accessible to the Hindi-speaking community. The primary motivations include:

**Unlocking Hindi Wikipedia**
Hindi Wikipedia contains vast amounts of information, but as unstructured prose intended for human readers. DBpedia transforms this repository into a structured, queryable knowledge graph by extracting information from infoboxes, tables, and categories.

**Centralized Query Endpoint**
Instead of requiring developers to build complex web scrapers for individual Wikipedia pages, the Hindi DBpedia provides a single, stable SPARQL endpoint. This centralized graph acts as a unified source of truth, dramatically simplifying data access for applications.

**Empowering Localized Applications**
By providing structured data in Hindi, this project enables developers to build culturally relevant applications for one of the world's largest language communities. Potential applications include:
- Intelligent chatbots with cultural context
- Localized search engine knowledge panels
- Academic research tools
- Recommendation systems tailored to Hindi-speaking audiences

### Case Study: Entity Exploration (Amitabh Bachchan)

I created a comprehensive demonstration focusing on a single well-defined entity to showcase fundamental query patterns:

**Entity Location and Retrieval**
```sparql
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX dbo: <http://dbpedia.org/ontology/>

SELECT ?person ?name
WHERE {
  ?person a dbo:Person .
  ?person foaf:name ?name .
  FILTER(LANG(?name) = "hi")
  FILTER(CONTAINS(?name, "अमिताभ बच्चन"))
}
LIMIT 10
```

**Property Traversal and Relationship Following**
```sparql
PREFIX dbo: <http://dbpedia.org/ontology/>
PREFIX dbr-hi: <http://hi.dbpedia.org/resource/>

SELECT ?parentDeathDate
WHERE {
  dbr-hi:अमिताभ_बच्चन dbo:parent ?parentURI .
  ?parentURI dbo:deathDate ?parentDeathDate .
}
```

### Advanced Query Capabilities

**Categorical and Aggregate Queries**
Demonstrated counting instances, assessing property availability, and performing complex aggregations:

```sparql
PREFIX dbo: <http://dbpedia.org/ontology/>
SELECT (COUNT(?s) as ?numberOfCities)
WHERE {
  ?s a dbo:City .
}
```

**Complex Relationship Traversal**
Showcased multi-step queries involving actors, movies, and directors:

```sparql
PREFIX dbo: <http://dbpedia.org/ontology/>
PREFIX dbr-hi: <http://hi.dbpedia.org/resource/>
PREFIX foaf: <http://xmlns.com/foaf/0.1/>

SELECT ?movieName ?directorName
WHERE {
  ?movie dbo:starring dbr-hi:अमिताभ_बच्चन .
  ?movie foaf:name ?movieName .
  OPTIONAL {
    ?movie dbo:director ?director .
    ?director foaf:name ?directorName .
  }
}
LIMIT 20
```

**Advanced Filtering and Data Types**
Demonstrated date filtering, regex pattern matching, and geographic coordinate retrieval:

```sparql
PREFIX dbo: <http://dbpedia.org/ontology/>
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>

SELECT ?personName ?birthDate
WHERE {
  ?person a dbo:Person ;
          dbo:birthDate ?birthDate ;
          foaf:name ?personName .
  FILTER (?birthDate > "1950-01-01"^^xsd:date)
}
ORDER BY ?birthDate
LIMIT 50
```

---

## Synthetic Data Generation and Model Benchmarking

### Data Generation Strategy

Building on the synthetic data generation work from Week 7, I adapted the script for OpenAI models and ran experiments with the 2.5 Flash model. The generation process yielded approximately 800 high-quality data points through:

**Multi-Strategy Approach**
- Structure-first generation (50% weight)
- Multi-relation generation (30% weight) 
- Targeted relation generation (20% weight)

**Quality Control Measures**
- Semantic concept grouping from benchie dataset
- Template-based sentence generation
- Programmatic validation and filtering
- Thought process inclusion for LM learning

### Model Benchmarking Results

**2.5 Flash Lite Performance**
After generating the synthetic data, I benchmarked the 2.5-flash-lite model on the Hindi benchie dataset:

```json
{
  "precision": 0.43,
  "recall": 0.43,
  "f1_score": 0.42999999999999994,
  "total_true_positives": 86,
  "total_false_positives": 114,
  "total_false_negatives": 114
}
```

**Key Observations**
- Balanced precision and recall (both at 43%)
- Significant improvement over previous baseline models
- Room for further optimization through better data quality
- Need for more diverse training examples

### Technical Challenges and Solutions

**Programmatic Validation Issues**
Attempted to implement programmatic validations and filtering mechanisms, but these only reduced true positives without improving overall performance. This suggests that the current validation approaches may be too restrictive for the task.

**Data Quality Improvements**
- Adapted script for OpenAI models successfully
- Generated approximately 800 high-quality data points
- Implemented chunk integrity maintenance
- Added comprehensive annotation guidelines

---

## Next Steps

### Immediate Actions
1. **DBpedia Mappings Access**: Requested access for updating DBpedia mappings via UI to improve data quality
2. **Model Optimization**: Continue refining synthetic data generation for better model performance
3. **Error Analysis**: Implement detailed error analysis for the 2.5 Flash model to identify improvement areas

### Future Directions
1. **Enhanced Data Generation**: Explore more sophisticated data generation strategies
2. **Model Finetuning**: Use the generated synthetic data for finetuning smaller models
3. **Performance Optimization**: Focus on improving precision while maintaining recall
4. **Integration Work**: Integrate improved models into the existing pipeline

---

## Conclusion

These two weeks have been highly productive, spanning academic research review, practical system demonstration, and data generation work. The paper review provided valuable insights into multilingual question answering approaches, while the SPARQL endpoint demonstration showcased the practical capabilities of our Hindi DBpedia chapter. The synthetic data generation work represents a significant step forward in improving model performance for Hindi information extraction tasks.

The balanced performance metrics from the 2.5 Flash Lite model (43% precision and recall) show promising progress, though there's still room for improvement through better data quality and model optimization strategies. 