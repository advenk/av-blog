---
title: "GSoC Week 6: IndIE Enhancement + Finetuning"
author: "Aditya Venkatesh"
pubDatetime: 2025-07-13T22:14:09.000Z
slug: gsoc-week-6
featured: true
tags: ["GSoC", "llmfinetuning", "dataprep", "informationextraction", "Week 6"]
description: "Week 6 of GSoC'25: Pushing the performance of our pipeline for Hindi information extraction."
---

## Introduction
This week, I try to push the performance of our system for hindi IE by implementing the suggestions of my mentors and also conduct an analysis of where our system makes mistakes so that we can improve upon it. I also draw out a plan for finetuning and evaluating gemma3 for our task. 

---
## Information Extraction

### Rules + LM Enhancement
In this experiment, I ran the system created in the last week with rules extraction followed by the LM component of generating more triples. I also updated the prompt to incorporate the two suggestions:
- Maintain chunk integrity
- I passed the dependency tree as triples instead of the previous structure. The MDT is now given as:
```
- ROOT: "सििद्ध होती है" (main action/predicate)
- ["कार्यरूप जगत को"] --(obj)--> ["सििद्ध होती है"]
- ["देखकर ही"] --(advcl)--> ["सििद्ध होती है"]
- ["शक्तिरूपी माया की"] --(nmod)--> ["सििद्ध होती है"]
```
- Also updated the prompt to refer to the MDT as just the dependency tree so as to not confuse the LM with unseen nouns from its training data

By running this system I was able to greatly increase the recall of the system. Following are the statistics: 

```
=== FINAL METRICS -> Rules + LM Enhancement ===
Total TP (True Positives): 186
Total FP (False Positives): 509
Total FN (False Negatives): 97
Recall 0.657243816254417
Precision 0.26762589928057556
F-score 0.3803680981595092
```

As we can see the numnber of TPs goes up to 186 from the previous best of 117! This is great news. Now we can try to build a pipeline for filtering as well. I tried running this along with the LM filtering implemented last time. However that fails miserably and the small LM gets confused due to the high number of triplets that it needs to clean and filter out of. As a result, if we run **rules + LM enhancement + filtering**, the number of TPs go down significantly. 

To counter this, I tried building a manual post-filter which would remove duplicates using fuzzy match and various methods. 

## LLM Finetuning

### Synthetic Data Generation
Using the gemini API, specifically the flash 2.0 model, we were able to generate synthetic data which we could use for our finetuning use case. We generated 8000 datapoints. 

### Finetuning
Since my local system runs on apple silicon, Initially I tried using the unsloth library on [google collab](https://colab.research.google.com/github/unslothai/notebooks/blob/main/nb/Gemma3N_(4B)-Conversational.ipynb#scrollTo=-Xbb0cuLzwgf) for finetuning gemma3-4b. However, I kept running into usage limits on the free tier. I then tried to use the [mlx library](https://github.com/ml-explore/mlx) for finetuning gemma locally. 

### Data Preparation
For finetuning gemma using mlx we have to prepare the data is *.jsonl (train, valid, test) files in the following format for instruction tuning:
```json
{"messages": [{"role": "system", "content": "Extract all subject-relation-object triplets from the given Hindi sentence. The output must be a valid JSON array of objects."}, {"role": "user", "content": "प्राथमिक शिक्षा बच्चों के भविष्य के लिए महत्वपूर्ण है, क्योंकि यह उच्च शिक्षा के लिए आधार बनाती है।"}, {"role": "assistant", "content": "[{\"subject\":\"प्राथमिक शिक्षा\",\"relation\":\"महत्वपूर्ण है\",\"object\":\"बच्चों के भविष्य के लिए\"},{\"subject\":\"प्राथमिक शिक्षा\",\"relation\":\"आधार बनाती है\",\"object\":\"उच्च शिक्षा के लिए\"},{\"subject\":\"आधार\",\"relation\":\"property\",\"object\":\"उच्च शिक्षा\"}]"}]}
```

Once this is done we can finetune the model using lora. 

### Finetuning Config
I finetuned the model using the following config:
```yaml
model: "google/gemma-3-4b-it"
train: true
data: 'mlx_data_chat'
seed: 0
batch_size: 2
iters: 6000
learning_rate: 1e-5
steps_per_report: 1000
save_every: 500
grad_checkpoint: true

lora_parameters:
  # The layer keys to apply LoRA to.
  # These will be applied for the last lora_layers
  keys: ["self_attn.q_proj", "self_attn.v_proj"]
  # keys: ["--all-linear-"]
  rank: 16
  alpha: 16.0
  scale: 1.0
  dropout: 0.05
```

### Results
After finetuning I ran this model through all the sentences in the Hindi benchie dataset. We obtain the following results:
```
Total TP (True Positives): 47
Total FP (False Positives): 235
Total FN (False Negatives): 146
Total Extractions: 282
Recall 0.24352331606217617
Precision 0.16666666666666666
F-score 0.19789473684210523
```

The results were very unsatisfactory. Upon further analysis it looks this could be because of the kind of data we trained on. Our data is not that good and hence our model doesn't learn very well to perform well on the hindi benchie dataset which expects relations and entities to be in a certain format. 

## Next Steps
- We will try to preprocess this data to be better. 
- In addition to this, we will consider alternate sources of data if possible. 

