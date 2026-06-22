# Source: https://freesolo.co/blog/people-search

# Building SOTA People Search

Published on October 29, 2025

Originally when Freesolo was first started when it was still called Linkd, there was one database for each school with approximately 10k profiles per school. That means that there is a lot of room for inefficiency. However, when we decided to build people search for the entire world and started working with a data provider with over 800M profiles, in order to keep the quality, the standard was a lot higher in terms of optimization.

In this blog, I’ll go over all the decisions behind our infra to create the SOTA people search engine.

Some interesting stuff:

- Scaling from local to global: how we went from per-school databases of ~10k profiles to a unified architecture indexing 800M+ people and 30M companies worldwide.
- From FAISS to OpenSearch: why dense-only retrieval broke down at scale and how we evolved toward a multi-layer retrieval stack (MySQL + OpenSearch).
- Cost vs. latency trade-offs: lessons from using BigQuery (great indexing, terrible cost per TB scanned) and the iterative optimizations that made real-time search feasible.
- Parallel ingestion at extreme scale: engineering a two-stage sequential pipeline that ingested 1.6 billion records in < 24 hours with atomic checkpointing and zero data loss.
- Agentic orchestration: building a prompt-to-SQL agent swarm capable of evaluating candidates and scraping additional signals with asyncio + Firecrawl.
- Embeddings at scale: self-hosting Qwen Embedding 0.6 on a distributed Runpod + NGINX cluster to generate hundreds of millions of embeddings at 12–15k EPS per GPU.
- Agentic chunking experiments: using LLMs to summarize each profile into multiple semantic facets.
- Hybrid retrieval (sparse + dense): evaluating Milvus BM25 + vector hybrid search, and why query-term explosion and large-scale union merges became prohibitively expensive at 800M profiles.
- Final architecture: a 8 TB OpenSearch index backed by a 25 TB Aurora MySQL store, achieving ~4 s end-to-end query latency for global-scale people search.

## MVP

![MVP](https://i.imgur.com/F8AyB4n.png)

The initial version of Linkd was built using a combination of Voyage embeddings and FAISS with a simple thresholding system. For small datasets (~1,000 profiles per school), the search quality was acceptable. However, as we scaled to larger datasets, two key issues emerged. First, the cost of generating and storing embeddings grew linearly with the number of profiles crawled, quickly becoming unsustainable. Second, even a 4,028-dimensional embedding wasn’t expressive enough to fully represent a person’s professional background. A single vector simply couldn’t capture the richness of multiple roles, skills, and experiences.

We also noticed semantic drift in ranking. For example, name-based searches like “David Shan” often prioritized sparse or incomplete profiles over those with more detailed information. Similarly, a search such as “FAANG engineer” would surface candidates who merely used to work at a FAANG company or were employed at adjacent firms like Microsoft, rather than those currently fitting the precise criteria. While embeddings offered decent recall, they lacked precision and context-awareness, which became a critical limitation once we began emphasizing accuracy and relevance at scale.

## Database + Agent

![Database + Agent](https://i.imgur.com/jNztE7u.png)

Afterward, we experimented with a prompt-to-SQL architecture by loading our data into BigQuery. The goal was to let an LLM interpret natural language queries and translate them into SQL, enabling structured filtering across hundreds of millions of profiles. To make this work, we built an agent swarm capable of handling high concurrency, scraping additional context from the web, and evaluating candidate relevance against user-defined criteria. We used Asyncio to orchestrate thousands of concurrent requests, while Firecrawl handled large-scale web enrichment for profiles with urls.

However, this approach quickly became cost-prohibitive. BigQuery’s pricing model charges by data scanned, and with roughly 200 million profiles (~1 TB of data), even a single query could cost upwards of $1 per search, which doesn’t scale for real-time applications. Despite its strong built-in inverted indexing, the economics and latency made BigQuery unviable for our use case.

We then migrated to MySQL to take greater control over indexing and query performance. While this significantly reduced cost, we found that MySQL’s native full-text indexing still wasn’t sufficient, search latency for 200 million profiles hovered around one minute per query, far above our real-time target. To address this, we introduced OpenSearch as a dedicated retrieval layer: MySQL became the system of record, storing full entity data, while OpenSearch indexed only the searchable fields. This hybrid architecture finally gave us the balance between speed, scalability, and precision that we had been searching for.

## Data Pipeline

![Data Pipeline](https://i.imgur.com/SnjKirt.png)

During the transition to OpenSearch, we made the decision to expand our dataset from 200 million profiles to over 800 million. This presented a significant engineering challenge: any ingestion pipeline would now need to be at least four times faster, or the team would be blocked for days without fresh data to test. To maintain iteration velocity, we set an ambitious goal, to design a pipeline that would complete in one day.

A second challenge emerged as we worked to enrich each profile with company data. Our datasets for people and companies existed in separate Parquet files, meaning the ingestion pipeline had to merge them before indexing. The resulting pipeline flow became: S3 → MySQL → OpenSearch, where MySQL served as the intermediate layer for joining and normalizing records before OpenSearch built the inverted index.

This dependency chain meant we couldn’t run the MySQL and OpenSearch ingestions in parallel — the OpenSearch step depended on fully written and joined data from MySQL. The sequential nature of this architecture introduced serious coordination and throughput challenges, forcing us to optimize every stage to sustain our one-day ingestion target.

### Stage 1: Parallel Data Ingestion to MySQL

![Step 1: Parallel Data Ingestion to MySQL](https://i.imgur.com/vbfnHDn.png)

We split the ingestion into two parallel streams - people and companies - each optimized differently:

People Ingestion Architecture:

1. 375 parallel worker processes - We pushed Python's multiprocessing to its limits, spawning hundreds of workers that could each handle their own parquet file independently
2. Subprocess isolation - Each worker ran as a separate Python process to avoid GIL contention and memory leaks
3. Atomic checkpointing - Workers appended completed files to a checkpoint file atomically, allowing us to resume from failures without data loss
4. Retry mechanism - Failed files got up to 3 retries before being logged to an error file for manual inspection

Company Ingestion Optimization:

1. 64 parallel workers with larger batch sizes (50,000 records)
2. Vectorized DataFrame transformations - Instead of row-by-row processing, we used pandas vectorized operations to transform entire columns at once
3. Bulk INSERT with ON DUPLICATE KEY UPDATE - MySQL's bulk insert with upsert semantics meant we could handle duplicates efficiently
4. Temporary file optimization - We used /mnt/dumps/ on high-speed NVMe drives for temporary parquet storage, falling back to system temp only when needed

### Stage 2: The MySQL to OpenSearch Challenge

The real complexity came in the second stage - joining 800M people records with company data and indexing into OpenSearch while the data was still flowing in:

#### Two-Phase Parallel Architecture:

1. Extraction Phase (MySQL → Parquet):
 - 128 parallel extraction workers each assigned a specific ID range using MySQL partition pruning
 - ID-based sharding - Each worker scanned a range like id > 100M AND id <= 107M, ensuring no overlap
 - LEFT JOIN optimization - We pre-joined people with companies in MySQL, avoiding the need for lookups during indexing
 - 50,000 record batches - Large enough for efficiency, small enough to fit in memory
2. Processing Phase (Parquet → OpenSearch):
 - File-based queueing system - We used atomic file operations (rename) to coordinate between extraction and processing stages
 - 24 concurrent processors - Each handling document transformation and bulk indexing
 - 10,000 document bulk operations - OpenSearch's sweet spot for bulk indexing performance
 - Continuous processing - Processors started working as soon as the first parquet files appeared, no waiting for extraction to complete

#### Critical Optimizations That Made It Possible

1. Memory Management:
 - Garbage collection every 10 batches to prevent memory bloat
 - Worker process recycling to avoid long-term memory leaks
 - Streaming processing - never loading the full dataset into memory
2. I/O Optimization:
 - NVMe SSDs for temporary storage with automatic fallback to system temp
 - Snappy compression for parquet files - fast compression with reasonable ratios
 - Cleanup of processed files to prevent disk exhaustion
3. Error Recovery:
 - Checkpoint files tracked both completed batches and continuous progress
 - Failed batches moved to a "failed" queue for retry or manual inspection
 - Graceful degradation - if one worker failed, others continued
4. Document Transformation Pipeline:
 - Pre-compiled regex patterns for date extraction
 - Cached JSON parsing and currency conversion
 - Efficient handling of nested structures (experiences, education, etc.)

#### The Result

With these optimizations, we achieved:

- 22 hours total ingestion time for 800M profiles
- ~10,100 profiles/second sustained throughput
- Zero data loss through atomic checkpointing
- 4-second end-to-end query latency on the final index

## Thoughts on Embeddings

![Embeddings](https://i.imgur.com/6d1HVoU.png)

Throughout this process, we kept revisiting the idea of embeddings since they had always seemed promising for improving semantic matching and relevance. Our earlier experiments, however, revealed a major limitation: a single embedding per profile wasn’t expressive enough to represent all of a person’s professional information. Profiles are inherently very complicated and compressing all that into one 4,000-dimensional vector loses too much nuance.

Still, with our dataset now expanded to over 800 million profiles, we decided to revisit embeddings to see whether scale could offset that loss in precision. Instead of relying on commercial APIs (which would’ve been prohibitively expensive at this scale), we self-hosted the Qwen Embedding 0.6 model on Runpod, giving us full control over throughput and cost.

To maximize GPU utilization, we configured each pod with a dedicated H100 GPU and deployed a lightweight FastAPI + PyTorch inference server on each one. On top of that, we used NGINX as a network load balancer, which routed incoming embedding requests across all active pods in a round-robin fashion. Each Runpod instance registered itself with NGINX using its internal IP, so the balancer could automatically distribute load evenly and retry failed pods without downtime. This setup allowed us to scale horizontally, thus adding new pods to the pool would immediately expand total throughput without requiring code changes or restarts.

Inside each pod, we built a custom dynamic batching system that grouped incoming embedding requests by token length rather than item count. This approach minimized padding overhead and fully saturated the GPU even under uneven workloads. Each pod processed around 40k–60k tokens per batch, and we maintained near 100% GPU utilization, sustaining roughly 12–15k embeddings per second per pod.

Qwen’s Matryoshka embedding design made it especially attractive for our use case, we were able to truncate embeddings to 512 dimensions, significantly reducing storage and memory usage without a major loss in semantic signal. In total, the distributed Runpod + NGINX cluster delivered the throughput we needed to handle hundreds of millions of profiles efficiently, at a fraction of the cost of hosted APIs.

![Embeddings](https://i.imgur.com/QPQ4yBF.png)

Despite the efficient setup, the results were underwhelming. The reduced dimensionality led to significant loss in granularity, and the smaller model struggled to capture deeper relational signals (e.g., role hierarchy, company prestige, or temporal job transitions).

We then explored agentic chunking, where an LLM summarizes each profile into 6–8 descriptive sentences before generating embeddings. The idea was that each sentence could represent a specific quality of a person and “criteria chunks” would later be matched against subcomponents of a user query. In practice, though, this required running multiple embedding similarity searches (one per criterion), then merging the top results. While theoretically elegant, it quickly became computationally infeasible at scale: memory usage ballooned, and merging thousands of similarity results added seconds of latency per query.

Afterwards, we tried a hybrid approach with Milvus by combining sparse and dense embeddings. However, with Milvus, the issue that we ran into is the fact that as the number of keywords increased, query latency grew significantly and exponentially. Moreover, the fusion stage between the sparse and dense retrieval results became extremely resource-intensive, since for a corpus of over 800 million profiles, the hybrid search requires retrieving and merging a large number of candidates from both the BM25 (sparse) and vector (dense) indexes. This meant that even with parallel execution, the union and re-ranking process ballooned in cost, making it difficult to sustain real-time latency at scale.

## Conclusion & What’s Next

![Conclusion](https://i.imgur.com/3Hk4rVL.png)

Building Freesolo’s people search engine has been a constant cycle of iteration: from small, school-level databases to a globally distributed 8 TB OpenSearch index and a 25 TB MySQL data warehouse powering more than 800 million profiles and 30 million companies. Each stage revealed new bottlenecks: not just in compute or storage, but in how data freshness, semantic depth, and infrastructure coordination ultimately determine real-world search quality.

Looking ahead, our focus is on both expanding data breadth and advancing retrieval intelligence. On the data side, we’re integrating new sources such as organizational charts, academic research via OpenAlex, and richer company datasets like org charts to add more structure and context to each profile. We’re also deepening our cross-platform graph, linking GitHub repositories, LinkedIn profiles, and other professional identifiers to better capture true professional relationships.

At the same time, we’re continuing to improve our in-house retrieval model, a small, RL-trained system built specifically for ranking and relevance optimization. The next major essay will dive into how we trained and reinforced this model, how it learns from user interactions, and how it helps Freesolo balance precision, recall, and latency across billions of data points.

Finally, we’re investing heavily in data freshness. Purpose-built crawlers and incremental sync pipelines are being deployed to keep profiles continuously up to date, ensuring that search results reflect the real-time professional world rather than a static dataset.

## Build your advantage for production AI.

 
[Speak to an Engineer](https://cal.com/freesolo/chat)

![Footer](https://freesolo.co/_next/image?url=%2F_next%2Fstatic%2Fmedia%2Ffooter.0f-08zjqrgtrd.png&w=3840&q=75&dpl=dpl_3MzvvU4hM5PSiZoNheicafFZjYDr)

![Freesolo](https://freesolo.co/_next/static/media/lockup.0ik4x--d92ipa.svg?dpl=dpl_3MzvvU4hM5PSiZoNheicafFZjYDr)

Copyright 2026 Linkd Inc. All rights reserved.

PRODUCT

[Blog](https://freesolo.co/blog)

COMPANY

[Careers](https://freesolo.co/contact) [Privacy Policy](https://freesolo.co/privacy) [Terms](https://freesolo.co/terms)

CONTACT

[Sales](https://freesolo.co/contact) [Support](https://freesolo.co/contact)

![Freesolo](https://freesolo.co/_next/static/media/lockup.0ik4x--d92ipa.svg?dpl=dpl_3MzvvU4hM5PSiZoNheicafFZjYDr)