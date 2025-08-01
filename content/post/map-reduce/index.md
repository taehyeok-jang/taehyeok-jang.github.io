---
title: MapReduce - Simplified Data Processing on Large Clusters
summary: MapReduce - Simplified Data Processing on Large Clusters
date: 2020-08-09
authors:
  - admin
tags:
  - data
---

## Background

Computation is arguably the very reason computers were invented. Unlike in the early days of computing, it is now possible to collect and store massive amounts of data. To process such data within a reasonable time frame, researchers proposed distributing it across multiple machines for parallel and distributed computation.

Among these approaches, MapReduce—invented at Google in the early 2000s introduced a programming model and implementation that enabled large-scale data to be processed across clusters of thousands of commodity machines. 

As shown in the paper, this distributed computing model was successfully adopted internally at Google and is now widely used in many large-scale data processing systems. While newer libraries such as Spark are now commonly used depending on the workload, a solid understanding of the MapReduce model remains highly beneficial.


## Abstract

MapReduce is a programming model and its corresponding implementation for processing and generating large data sets. Programmers express computations through a map function that processes input key/value pairs into intermediate key/value pairs, and a reduce function that merges all intermediate values associated with the same intermediate key. As shown in the paper, many real-world tasks can be effectively expressed in this model.

Programs written in this functional style are automatically parallelized and executed on large clusters of commodity machines. The MapReduce runtime handles numerous details essential in distributed environments: partitioning input data, scheduling tasks across machines, managing failures, and coordinating inter-machine communication. This abstraction allows programmers with little to no experience in distributed or parallel systems to efficiently harness large-scale computing resources.

## 1. Introduction

At Google, large-scale computations were routinely written to process massive volumes of raw data—such as crawled documents and web logs—into derived data structures like inverted indices and link graphs. However, as input sizes grew, these computations had to be distributed across thousands of machines. This introduced complexity that distracted from the simplicity of the original computational task:

- Parallelizing computations
- Distributing data across machines
- Handling machine failures

To mitigate these complexities, Google introduced a new abstraction that hides details of distributed execution, allowing programmers to focus on the core computation. Inspired by the map and reduce primitives from functional programming languages like LISP, they realized that most of their computations could be expressed using these two operations; map and reduce. Furthermore, this functional approach enabled straightforward re-execution, which served as the primary mechanism for parallelization and fault tolerance.

The paper’s main contributions are:
- A simple yet powerful interface that supports parallel and distributed computation at scale.
- A high-performance implementation that runs on large clusters of commodity PCs.

## 2. Programming Model

MapReduce operates on input key/value pairs and produces output key/value pairs. Users of the MapReduce library define two primary functions:

**Map()** takes an input key/value pair and produces a set of intermediate key/value pairs. The MapReduce runtime then groups all intermediate values associated with the same key and forwards them to the reduce function.

**Reduce()** takes an intermediate key and a set of values associated with that key. In the actual implementation, values are provided as iterators to support large datasets. The function merges and processes these values to generate the final output.

### 2.1 Example

A classic example of the MapReduce model is word count:

```
map(String key, String value):
    // key: document name
    // value: document contents
    for each word w in value:
        EmitIntermediate(w, "1");

reduce(String key, Iterator values):
    // key: a word
    // values: a list of counts
    int result = 0;
    for each v in values:
        result += ParseInt(v);
    Emit(AsString(result));
``` 

### 2.2 Types

While the above pseudocode uses strings for input and output, the model can be abstracted as:

```
map (k1,v1) → list(k2,v2)
reduce (k2,list(v2)) → list(v2)
```

### 2.3 More Examples

Several other programs can be elegantly expressed in the MapReduce model:

- Distributed Grep
- Count of URL Access
- Reverse Web-Link Graph
- Term-Vector per Host
- Inverted Index
- Distributed Sort

## 3. Implementation

The MapReduce interface can be implemented in various environments, from small shared-memory machines to large clusters. This paper focuses on Google’s implementation on clusters of commodity machines connected via switched Ethernet. Though the paper was written in 2004, the fundamental challenges remain valid today, as network bandwidth is still a limited and valuable resource.

### 3.1 Execution Overview

Input data is split into M partitions processed by M map tasks. Intermediate key space is partitioned into R reduce tasks. A typical partition function is hash(key) mod R.

**Execution steps:** 

1. Input files are split into M pieces (typically 16–64 MB).
2. One program instance acts as the master; others are workers. The master assigns M map tasks and R reduce tasks to idle workers.
3. Map workers parse input splits into key/value pairs, process them using the map function, and buffer the intermediate pairs in memory.
4. These buffered pairs are periodically written to local disk, partitioned into R regions. The master keeps track of these locations.
5. Reduce workers fetch intermediate data from map workers via RPC, sort it by key, and group values with the same key.
6. The reduce function is applied to each group, and the output is written to final output files.
6. When all tasks complete, the master notifies the user program.

### 3.2 Master Data Structures

The master tracks the state (idle, in-progress, completed) of all tasks and maintains the location info of intermediate data.

### 3.3 Fault Tolerance

**Worker Failure:**
Workers are pinged periodically. On failure, their tasks are reassigned. Intermediate results are recomputed if needed.

**Master Failure:**
Checkpointing allows restarting from the last saved state, though the current implementation aborts the job and relies on client-side retries.

**Semantics:**
If map/reduce functions are deterministic, results are equivalent to a sequential, fault-free execution.

### 3.4 Locality

To save bandwidth, MapReduce prefers to schedule map tasks on machines that store the input data locally or are nearby.

### 3.5 Task Granularity

Choosing large values for M and R allows better load balancing and faster recovery. Practical limits arise from scheduling and metadata overhead.

### 3.6 Backup Tasks

To mitigate the effect of stragglers, the system launches backup executions of slow tasks as the job nears completion. This significantly reduces job latency with minimal resource overhead.

## 4. Refinements

See paper for additional implementation refinements.

## 5. Performance

### 5.1 Cluster Configuration

Experiments were run on a cluster of ~1800 machines, each with:
- 2GHz Xeon CPU, 4GB RAM
- 2x160GB IDE disks
- Gigabit Ethernet
- RTT < 1ms between any two machines

### 5.2 Grep

A grep job searched for a rare 3-character pattern across 10^10 100-byte records (~1TB). M = 15,000; R = 1. Peak utilization involved 1764 workers, with total runtime of ~150s.

### 5.3 Sort

Sort benchmark (similar to TeraSort) sorted 1TB of data. M = 15,000, R = 4,000. The map function extracted 10-byte keys, and the reduce function was the identity. Output was 2-way replicated in GFS. Graphs showed high input rate due to locality, and output bandwidth dominated by replication overhead.

### 5.4 Backup Tasks

Disabling backup tasks resulted in 44% longer execution time due to stragglers.

### 5.5 Machine Failures

Simulating the failure of 200 workers caused only a 5% increase in job time, thanks to rapid rescheduling.

## 6. Experience

Since its early prototype in February 2003, MapReduce has been applied across a wide range of Google workloads:

- Large-scale machine learning
- Clustering (e.g., for Google News)
- Popular query reports (e.g., Google Zeitgeist)
- Geographic extraction for local search
- Large-scale graph computations

## 7. Related Work

MapReduce draws inspiration from various prior systems:
- Abstractions for parallelization
- Locality-aware scheduling
- Backup execution
- In-house cluster management
- Distributed queue-based programming

## 8. Conclusion

MapReduce has been successfully adopted within Google for many large-scale data processing tasks. Its success stems from:

- Ease of use, even for non-experts in distributed systems
- Broad applicability to different tasks
- An implementation that scales to thousands of machines

Key takeaways include:
- A restricted programming model simplifies parallelization and fault-tolerance
- Network bandwidth is invaluable; locality matters
- Redundant execution improves reliability and reduces job latency

## References

Dean, J., & Ghemawat, S. (2004). MapReduce: Simplified Data Processing on Large Clusters. OSDI'04: Sixth Symposium on Operating Systems Design and Implementation, 137–150.