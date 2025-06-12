---
title: "GSoC Weeks 1-2: Bootstrapping DBpedia Hindi Extraction"
author: "Aditya Venkatesh"
pubDatetime: 2025-06-11T15:37:07.000Z
slug: gsoc-week-1
featured: true
tags: ["GSoC", "DBpedia", "Week 1", "Hindi"]
description: "Weeks 1 & 2 of my GSoC'25 journey: setting up the DBpedia Information Extraction Framework and Neural Extraction Framework for the Hindi chapter, overcoming roadblocks, and first learnings."
---

> "The beginning is the most important part of the work." — Plato (and every developer ever, after a fresh `git clone`)

## Introduction

The first two weeks of my GSoC'25 journey with DBpedia have been a whirlwind of setup, debugging, and learning. My project focuses on the Hindi chapter, aiming to streamline and enhance the extraction of structured knowledge from Hindi Wikipedia using both the classic DBpedia framework and modern neural methods.

This post is a logbook of my experience: the hurdles, the fixes, and the small wins that make open-source work so rewarding.

---

## What is the DBpedia Information Extraction Framework (DIEF)?

The [DBpedia Information Extraction Framework (DIEF)](https://github.com/dbpedia/extraction-framework) is the backbone of DBpedia's knowledge extraction pipeline. It parses Wikipedia dumps, applies mapping rules, and outputs structured data (subject-predicate-object triples). Compiling and running DIEF to locally for the Hindi configuration was the first step to meaningful contributions.

<!-- ![Placeholder: DIEF Pipeline Diagram](/av-blog/images/dief-arch.png) -->

## What is the Neural Extraction Framework (NEF)?

The [Neural Extraction Framework (NEF)](https://github.com/dbpedia/neural-extraction-framework) leverages machine learning models for tasks like entity recognition, coreference resolution, and entity linking—pushing DBpedia's extraction capabilities beyond rule-based systems.

<!-- ![Placeholder: NEF Model Overview](/av-blog/images/nef-overview.png) -->

---

## Weeks 1-2: Setup, Debug, Repeat

### DIEF

My first challenge was getting DIEF up and running. I initially tried running the extraction framework directly, which led me down a path of cryptic errors: missing property files, incomplete configurations, and issues with the wiki dump file itself.

After a few days of struggle, a lifeline from my mentor: "use the [`marvin-config`](https://github.com/dbpedia/marvin-config) repository". This is designed for easy deployment and abstracts away much of the manual hurdles. It still required some tweaking for the Hindi chapter, including adding download and extraction configs, updating paths, and adjusting the download process to use our updated Hindi mappings. You can see all the necessary changes in [this commit](https://github.com/dbpedia/marvin-config/commit/bafb9a4efcf3d6cead757f63498638bc22201cfb).

Even with `marvin-config`, there were a couple of roadblocks:

**1. The Malformed Wiki Stats CSV**

The framework downloads a CSV of wiki statistics from `wikistats.wmcloud.org`. Unfortunately, a malformed line for the "Tai Nüa" language was breaking the parser.

```
375,"Tai N&#252;a",tdd,1394,340,,5936,581,1,"2025-06-10 00:12:14","..."
```

The fix was to download this file manually, correct the problematic line, and point the framework to my local, corrected version by changing a hardcoded URL in `WikiInfo.scala`. This is a temporary workaround. I plan to discuss with my mentors about making this URL configurable and getting the source file fixed.

**2. Querying the Extracted Data with Virtuoso**

Once the extraction pipeline ran successfully, I needed a way to query the generated data. This is where Virtuoso comes in, providing a SPARQL endpoint. Setting it up for local data was straightforward: I just had to comment out the `COLLECTION_URI` variable in the config and place my compressed `.bz2` output files into the `downloads` directory. Success! I could finally run SPARQL queries and see the Hindi triples I had extracted.

<!-- <img src="/av-blog/images/dief-sparql-success.png" alt="SPARQL query results showing extracted Hindi data" style="width: 70%;" /> -->

### NEF: Chasing Model Files

Setting up the Neural Extraction Framework (NEF) was a different kind of struggle, more about fixing dependencies and file paths than legacy code. The process involved:

-   **Model Downloads:** The download script in the repo had issues (duplicate `tar` commands, Google Drive links not working with `wget`). I switched to `gdown` for downloads and fixed the extraction commands.
-   **Dependency Hell:** Several Python packages were missing from `requirements.txt`. After some trial and error, I added everything needed to run all the modules.
-   **File Locations:** The code expected model files in very specific directories. I patched the code to look in the right places to match the downloaded structure.
-   **Manual Fixes:** For entity linking, I had to patch the Fairseq model loader to allow non-weights-only checkpoints.

After all this, the NEF demo finally ran—coreference, chunking, and entity linking all working. The feeling? Like finally seeing the output of a long-running `make` command succeed.

You can see the PR for streamlining this repo [here](https://github.com/dbpedia/neural-extraction-framework/pull/19).

<!-- ![Placeholder: NEF Demo Screenshot](/av-blog/images/nef-demo.png) -->

---

## Learnings and Reflections

These first two weeks have been a steep learning curve. My key takeaways:

-   **Legacy code is a teacher:** Every error message is a clue. Reading stack traces and config files taught me more than any tutorial could.
-   **Community matters:** If not for my mentor, I would have been stuck for way longer. Open source is a team sport, and seeking help is part of the process.
-   **Automation is key:** Manual fixes work for now, but my next step is to script these setups to help future contributors (and my future self).
-   **Deep learning frameworks are picky:** Model paths, config files, and dependency versions all matter. Consistency is everything.

---

## What's Next?

Over the next week, my goals are:

-   Work on LLM optimization by running the Gemma 1B model.
-   Evaluate Gemma's performance against our existing Mistral implementation for extraction tasks.

Stay tuned for more technical deep-dives, benchmarks, and (hopefully) fewer manual fixes!

<!-- ![Placeholder: Team Call Screenshot](/av-blog/images/kickoff-meet.jpeg)  -->