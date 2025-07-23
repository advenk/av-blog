---
title: "GSoC Week 7: Synthetic Data Generation and Performance Testing of Hindi Chapter"
author: "Aditya Venkatesh"
pubDatetime: 2025-07-23T09:46:29.000Z
slug: gsoc-week-7
featured: true
tags: ["GSoC", "syntheticdatagen", "deployment", "sparql", "Week 7"]
description: "Week 7 of GSoC'25: Rewriting the synthetic data generation script and performance testing of SPARQL endpoint"
---

## Introduction

As last week's results of the finetuned model were unsatisfactory, we decided to rewrite the synthetic data generation script to get better data that aligns with our goals (representative of the benchie dataset). In addition to this, we got temporary server access from the DBpedia team for testing and benchmarking the Hindi SPARQL endpoint. 

---

## Synthetic Data Generation
The primary problem with our previously generated data was that it was not representative of the benchie dataset. That is why when we finetuned the base model (gemma3-4b), it learned well and performed well on the test split with a loss of 0.484 and test perplexity of 1.623, but it didn't perform well on the benchie dataset (recall of 24%). 

### Data Generation Strategy

1. I extracted all relations from the benchie dataset and grouped all of these into semantic concepts as such:
```json
SEMANTIC_CONCEPTS = {
    "Becoming & Appointment": ["बने", "बन चुके हैं", "नियुक्त हुई", "निदेशक बने", "घोषित किया गया", "एक प्रमुख हस्ती बन चुके हैं", "आजाद हुआ"],
    "Initiation & Beginning": ["शुरू की", "शुरू हुई", "आरम्भ हुई", "सुनवाई शुरू हुई", "आधारशिला रखी गयी थी", "बोलीवुड करियर की शुरुआत की"],
    ...
}
```


2. We also extracted sentence structure templates from all the benchie sentences as such:
["A sentence stating a contribution to a field (which is a list of two items joined by 'और'), using a comparison with another person ('...के समान').",
"A sentence explaining the usage of a word ('... प्रयोग में आता है'), where the entity it refers to is described by a participle phrase ('... बनाने वाली')."
...
]


Now we have three sub-strategies: `structure_first`, `multi_relation`, and `targeted_relation`.
- In `structure_first`, we prompt the LLM to generate a sentence using a template of a random structure and give it one of the random relations to incorporate in that. 
- In `multi_relation`, we prompt the LLM to use two of the random relations from two **different** semantic concepts. 
- In `targeted_relation`, we prompt the LLM to use randomly one of the relations to generate a sentence. 

The script runs in a loop and picks which strategy to execute based on a weight assigned to it. We assign 50% weight to `structure_first`, then 30% to `multi_relation`, and 20% to `targeted_relation`. 
This ensures the generated data is representative of our final Hindi benchie dataset. We also added instructions to output a "thought_process" field which our small LM can learn from for extracting relations. The prompt also has specific annotation guidelines for the LLM to adhere to benchie standards. 

You can find the whole script [here](https://github.com/advenk/neural-extraction-framework/blob/gsoc25_main/GSoC25_H/llm_IE/finetuning/synthetic_data_gen_2.py).

---

## Test Deployment and Testing

### Server Setup

First things first, the server was set up with Ubuntu 24.x. SSH access was added and then I added the required dependencies like Docker, pip, and Python. 

### Creating the Docker Image
Once the server was set up, I made the changes for simple deployment of the Hindi chapter via Docker image. I set up the Dockerfile to set the environment variables, copy the RDF data into the container, copy the data loading and startup script, and finally expose the ports. 

Then I built and ran the image locally for testing.
```
docker build -t hindi-dbpedia-sparql:latest .
docker run -d -p 8890:8890 -p 1111:1111 --name hindi-sparql-test hindi-dbpedia-sparql:latest
```

There were a few hiccups with the data loading script which were resolved, and then the image was pushed to my Docker Hub after using a multi-platform build to support both AMD and ARM architecture.
```
docker buildx build --platform linux/amd64,linux/arm64 \
  -t 42bitstogo/hindi-dbpedia-sparql:latest \
  --push .
```

On the server I deployed this image using:
```
docker run -d -p 8890:8890 -p 1111:1111 --name hindi-sparql 42bitstogo/hindi-dbpedia-sparql:latest
```

### Performance Testing
The server was tested for performance with varying levels. I simulated hitting the server with multiple requests at different levels of concurrency.

#### Simple Query Performance
3 different simple queries:
- SELECT (COUNT(*) AS ?count) WHERE { ?s ?p ?o }
- SELECT (COUNT(DISTINCT ?s) AS ?count) WHERE { ?s ?p ?o }
- SELECT ?s ?p ?o WHERE { ?s ?p ?o } LIMIT 10
Results: 0.7-0.9ms response times

#### Complex Query Performance
- Aggregation query with GROUP BY and ORDER BY
- Result: 0.8ms response time

#### Quick Load Test
- 5 concurrent requests, 10 iterations
- Result: 20ms average, 100% success rate


#### Comprehensive Python Performance Testing 
```python
queries = [
    "Total Triples Count",
    "Distinct Subjects", 
    "Distinct Properties",
    "Person Entities",
    "Geographic Data"
]
```
**Purpose:** Test fundamental data access patterns  
**Results:** 0.02s - 1.12s response times  
**Coverage:** Count queries, entity queries, geographic data



```python
complex_queries = [
    "Multi-Join Query",
    "Aggregation Query", 
    "Text Search"
]
```
**Purpose:** Test advanced SPARQL features  
**Results:** 0.01s - 0.04s response times  
**Coverage:** Joins, aggregations, text search with Hindi filters


#### Concurrent Load Testing

```python
def test_concurrent_load(self, num_threads=10, duration=60):
    # 10 threads running for 60 seconds
    # Continuous query execution
```
**Purpose:** Measure sustained throughput under load  
**Results:** 23,515 queries, 391.92 QPS, 25.5ms average response time  
**Coverage:** Multi-threaded sustained load

#### Memory Usage Testing
```python
def test_memory_usage(self):
    # Monitor memory before/after queries
    # Test with different query sizes
```
**Purpose:** Measure memory efficiency and stability  
**Results:** 1.95GB baseline, 1.97GB peak, 0.01GB increase  
**Coverage:** Memory leak detection, resource efficiency


#### Advanced Load Testing (load_test.sh)
**Progressive Concurrency Testing**
```python
for concurrency in [1, 5, 10, 20, 50]:
    # 100 requests per concurrency level
    # Measure success rate, response times
```
**Purpose:** Progressive concurrency scaling analysis  
**Results:** 100% success rate across all levels  
**Response Times:** 12ms → 16ms → 25ms → 45ms → 64ms  
**Coverage:** Concurrency limits and scaling characteristics


#### Resource Monitoring During Load
Background monitoring during 60-second heavy load test with 20 workers, continuous requests

**Purpose:** Monitor system resources under sustained load  
**Results:** 23,488 requests in 60s, 391.47 QPS  
**Coverage:** CPU, memory, network usage during peak load

### Final Results
**Comprehensive Test Coverage Matrix**

| Test Category | Test Type | Duration | Concurrency | Metrics Measured |
| :---- | :---- | :---- | :---- | :---- |
| **Connectivity** | Single Request | \<1s | 1 | Response time, HTTP status |
| **Basic Performance** | Simple Queries | \<5s | 1 | Query response times |
| **Complex Performance** | Advanced Queries | \<10s | 1 | Join/aggregation performance |
| **Quick Load** | Concurrent Requests | \<30s | 5 | Success rate, response times |
| **Sustained Load** | Continuous Load | 60s | 10 | Throughput, QPS, response times |
| **Concurrency Scaling** | Progressive Load | Variable | 1-50 | Scaling characteristics |
| **Resource Monitoring** | System Resources | 60s | 20 | CPU, memory, I/O usage |
| **Memory Testing** | Memory Usage | \<30s | 1 | Memory efficiency, leaks |

#### **Key Performance Metrics Collected**

**Response Time Metrics**

- **Baseline:** 12ms (single request)  
- **Simple Queries:** 0.7-0.9ms  
- **Complex Queries:** 0.7-0.8ms  
- **Concurrent Load:** 25.5ms (10 threads)  
- **Peak Load:** 63.5ms (50 concurrent users)

**Throughput Metrics**

- **Peak QPS:** 391.92 requests/second  
- **Total Queries:** 23,515 (60-second test)  
- **Success Rate:** 100% across all tests

**Resource Utilization**

- **CPU Usage:** 0% (idle) → 750% (peak load)  
- **Memory Usage:** 1.3GB stable  
- **Memory Efficiency:** 2% of available 62GB RAM

**Scalability Metrics**

- **Concurrency Scaling:** Linear response time increase  
- **Maximum Tested:** 50 concurrent users  
- **Performance Degradation:** Predictable and acceptable

All of the code and results in detail are available on [GitHub](https://github.com/advenk/virtuoso-sparql-endpoint-quickstart/tree/gsoc25_hindi_chapter).

---

## Next Steps

- Continue working on finetuning once we have the data collected from the synthetic data generation script.
- Upon receiving the permanent server we can deploy the Hindi chapter sparql endpoint.
