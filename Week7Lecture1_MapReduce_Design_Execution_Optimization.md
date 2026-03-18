# Big Data Analytics (BDA Spring 2026)
## Week 7, Lecture 1: The MapReduce Programming Model - Design, Execution and Optimization

> MapReduce is not the dominant processing framework in 2026. Apache Spark has largely replaced it. You will likely never write a production MapReduce job in your career. We study it because it is the conceptual foundation of every distributed data processing framework that followed. Understanding MapReduce deeply means understanding the DNA of Spark, Flink, and Kafka Streams.

---

## Table of Contents

1. [Origins and Philosophy](#1-origins-and-philosophy)
2. [MapReduce Data Model - Keys and Values](#2-mapreduce-data-model---keys-and-values)
3. [Execution Model - Complete Deep Dive](#3-execution-model---complete-deep-dive)
4. [Fault Tolerance - MapReduce's Superpower](#4-fault-tolerance---mapreduces-superpower)
5. [Optimization - Practical Tuning](#5-optimization---practical-tuning)
6. [Multi-Step MapReduce - Chaining Jobs](#6-multi-step-mapreduce---chaining-jobs)
7. [MapReduce in 2026 - Legacy and Relevance](#7-mapreduce-in-2026---legacy-and-relevance)

---

## 1. Origins and Philosophy

### The 2004 Google Paper

MapReduce was introduced in a landmark 2004 paper: **"MapReduce: Simplified Data Processing on Large Clusters"** by Jeffrey Dean and Sanjay Ghemawat. One of the most cited and influential papers in computer science history.

**The core insight:** most of Google's large-scale computations - building the web index, computing PageRank, processing web logs - could be expressed as two simple operations borrowed from functional programming: `map` and `reduce`.

```mermaid
flowchart LR
    BEFORE[Before MapReduce] --> B1[Deep distributed systems expertise needed]
    BEFORE --> B2[Handle node failures manually]
    BEFORE --> B3[Partition data manually]
    BEFORE --> B4[Coordinate workers manually]

    AFTER[After MapReduce] --> A1[Write map function]
    AFTER --> A2[Write reduce function]
    AFTER --> A3[Framework handles everything else]

    style BEFORE fill:#c1121f,color:#fff,stroke:#c1121f
    style AFTER fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

Any programmer who could write a map function and a reduce function could process petabytes of data on a thousand machines.

---

### The Functional Programming Roots

```python
# Functional MAP: apply function to every element
numbers = [1, 2, 3, 4, 5]
squares = list(map(lambda x: x * x, numbers))
# squares = [1, 4, 9, 16, 25]

# Functional REDUCE: combine all elements into one value
from functools import reduce
total = reduce(lambda acc, x: acc + x, numbers)
# total = 15
```

MapReduce scales these to distributed systems:

| Concept | Single Machine | MapReduce Distributed |
|---------|---------------|----------------------|
| Map | Apply function to list in memory | Apply to billions of records across thousands of machines |
| Reduce | Combine list in memory | Combine across network-distributed intermediate results |
| Parallelism | Sequential on one core | Trivially parallel - each map is independent |

**Why parallelism is trivial:** each map invocation operates on exactly one input record and is completely independent of every other map invocation. No coordination needed. Run one per record across any number of machines simultaneously.

---

## 2. MapReduce Data Model - Keys and Values

Everything in MapReduce is expressed as **key-value pairs**. This is the universal data model.

```mermaid
flowchart TD
    INPUT[Input - HDFS files] --> KV1[InputFormat splits into key-value pairs]
    KV1 --> MAP[Map function - one input KV pair]
    MAP --> INTER[Emits zero or more intermediate KV pairs]
    INTER --> SHUFFLE[Shuffle - group all values with same key]
    SHUFFLE --> REDUCE[Reduce function per key with all values]
    REDUCE --> OUTPUT[Output - written to HDFS as KV pairs]

    style INPUT fill:#1d3557,color:#fff,stroke:#1d3557
    style MAP fill:#457b9d,color:#fff,stroke:#457b9d
    style SHUFFLE fill:#e07c24,color:#fff,stroke:#e07c24
    style REDUCE fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style OUTPUT fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

| Phase | Key | Value |
|-------|-----|-------|
| Input (TextInputFormat) | Byte offset in file | Line of text |
| Map output | Word emitted by user | Integer 1 |
| Reduce input | One unique word | Iterator over all 1s for that word |
| Reduce output | Word | Final count |

The key-value model is deliberately minimal. The constraint forces all computation to be expressed as transformations and aggregations on key-value pairs - this is exactly what enables automatic distribution and fault tolerance.

---

## 3. Execution Model - Complete Deep Dive

### Phase 0 - Job Submission and Input Splitting

```mermaid
flowchart TD
    CLIENT[Client submits job to YARN] --> AM[YARN launches ApplicationMaster]
    AM --> SPLIT[InputFormat - one InputSplit per block]
    SPLIT --> NUMMAP[Map tasks = number of InputSplits]
    AM --> REQ[Request Map containers on data DataNodes]
    REQ --> LOCALITY[Data locality - tasks run where data lives]
    AM --> NUMRED[Reducer count = job.setNumReduceTasks N]

    style CLIENT fill:#1d3557,color:#fff,stroke:#1d3557
    style AM fill:#457b9d,color:#fff,stroke:#457b9d
    style LOCALITY fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style NUMRED fill:#e07c24,color:#fff,stroke:#e07c24
```

---

### Phase 1 - The Map Phase

```mermaid
flowchart TD
    READ[Step 1 - RecordReader reads local block] --> APPLY[Step 2 - Map function per input KV pair]
    APPLY --> EMIT[Map emits intermediate KV pairs]
    EMIT --> BUFFER[Step 3 - Pairs collected in 100 MB buffer]
    BUFFER --> SPILL{Buffer at 80 percent?}
    SPILL -->|Yes - background thread| PART[Partition by hash of key mod numReducers]
    PART --> SORT[Sort within each partition by key]
    SORT --> DISK[Spill sorted partitioned data to disk]
    SPILL -->|Map complete| MERGE[Merge spill files into one map output]
    MERGE --> COMB[Step 4 - Optional Combiner runs locally]

    style READ fill:#1d3557,color:#fff,stroke:#1d3557
    style BUFFER fill:#457b9d,color:#fff,stroke:#457b9d
    style PART fill:#e07c24,color:#fff,stroke:#e07c24
    style SORT fill:#e07c24,color:#fff,stroke:#e07c24
    style COMB fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

**Word Count map example:**

```
Input:  (offset=0,  "the quick brown fox")
Output: ("the",1), ("quick",1), ("brown",1), ("fox",1)

Input:  (offset=20, "the fox jumped")
Output: ("the",1), ("fox",1), ("jumped",1)
```

**Partitioning formula:**

$$\text{partition} = \text{hash}(key) \mod \text{numReducers}$$

All pairs for Reducer 0 go to partition 0, all for Reducer 1 go to partition 1, and so on. This guarantees that all values for any given key reach exactly one reducer.

**Why sort within each partition?**

$$\text{Sorting by key} \Rightarrow \text{identical keys are adjacent} \Rightarrow \text{linear scan suffices to group values}$$

Sorting complexity is O(N log N). The alternative (hashing) is O(N) but requires more memory and produces unordered output. MapReduce chose sorting for predictable, bounded memory usage and sorted output useful for downstream operations.

---

### The Combiner - Local Pre-Aggregation

```mermaid
flowchart LR
    WITHOUT[Without Combiner] --> W1[Emit all pairs to shuffle]
    W1 --> W2[the-1 the-1 the-1 fox-1 fox-1]
    W2 --> W3[5 pairs transferred over network]

    WITH[With Combiner] --> C1[Run reduce locally on map output]
    C1 --> C2[the-3 fox-2]
    C2 --> C3[2 pairs transferred over network]

    style WITHOUT fill:#c1121f,color:#fff,stroke:#c1121f
    style WITH fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

The Combiner is a local Reducer that runs on each Map task before the shuffle. For a corpus where "the" appears millions of times per block, the Combiner reduces millions of `("the", 1)` pairs to one `("the", N)` pair per Map task.

**Combiner constraint - must be commutative and associative:**

$$\text{Sum: } f(f(3,5), 7) = f(3, f(5,7)) \quad \checkmark \text{ Safe}$$

$$\text{Average: } avg(avg(3,5), 7) = avg(4, 7) = 5.5 \neq avg(3, 5, 7) = 5.0 \quad \times \text{ Not safe}$$

---

### Phase 2 - The Shuffle Phase

The shuffle is the most network-intensive phase. It is where the distributed nature of computation is most visible.

**What is the shuffle?** Moving each Reducer's share of the map output (Partition K) from all Map tasks to Reducer K.

```mermaid
flowchart LR
    MT1[Map Task 1 output] --> PART0_1[Partition 0]
    MT1 --> PART1_1[Partition 1]
    MT1 --> PART2_1[Partition 2]

    MT2[Map Task 2 output] --> PART0_2[Partition 0]
    MT2 --> PART1_2[Partition 1]
    MT2 --> PART2_2[Partition 2]

    PART0_1 --> RED0[Reducer 0 - A to H keys]
    PART0_2 --> RED0
    PART1_1 --> RED1[Reducer 1 - I to P keys]
    PART1_2 --> RED1
    PART2_1 --> RED2[Reducer 2 - Q to Z keys]
    PART2_2 --> RED2

    style MT1 fill:#1d3557,color:#fff,stroke:#1d3557
    style MT2 fill:#1d3557,color:#fff,stroke:#1d3557
    style RED0 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style RED1 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style RED2 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

**Shuffle mechanics:**

| Sub-phase | What Happens |
|----------|-------------|
| Fetch | Reduce tasks send HTTP requests to Map task nodes, fetching their partition as soon as Map tasks complete (pipelined - does not wait for all maps to finish) |
| Merge | As partitions arrive, Reduce task merges them maintaining sorted order using N-way merge sort |
| Spill | If merged data exceeds memory, intermediate merged results are written to local disk |

**The shuffle is the bottleneck.** In a large job it transfers terabytes across the cluster network. Mitigation strategies:

| Strategy | Mechanism | Benefit |
|---------|-----------|---------|
| Combiner | Reduce pairs before shuffle | Cuts shuffle volume by 10x or more |
| Map output compression | Snappy compress before transfer | Cuts bytes over network by 3-5x |
| Rack-aware scheduling | Prefer intra-rack data locality | Reduces cross-rack bottleneck |

---

### Phase 3 - The Reduce Phase

```mermaid
flowchart TD
    MERGED[All partitions fetched and merged] --> GROUP[Group consecutive identical keys]
    GROUP --> INVOKE[Invoke reduce with key and value iterator]
    INVOKE --> WRITE[Write output KV pairs via OutputFormat]
    WRITE --> FILES[part-r-00000, part-r-00001 per reducer]

    style MERGED fill:#1d3557,color:#fff,stroke:#1d3557
    style INVOKE fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style FILES fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

**Word Count reduce example:**

```
Input:  ("brown",  [1])     → Output: ("brown",  1)
Input:  ("fox",    [1, 2])  → Output: ("fox",    3)
Input:  ("jumped", [1])     → Output: ("jumped", 1)
Input:  ("quick",  [1])     → Output: ("quick",  1)
Input:  ("the",    [3, 1])  → Output: ("the",    4)
```

"fox" has values [1, 2]: one direct 1 from Map Task 1, one 2 from Map Task 2 after the Combiner summed two 1s. Reducer correctly sums to 3.

**Output is sorted per partition but NOT globally sorted.** For global sort, use `TotalOrderPartitioner` which samples the key distribution and assigns key ranges such that Partition 0 < Partition 1 < ... This is what the TeraSort benchmark uses.

---

### Complete Data Flow

```mermaid
flowchart LR
    HDFS_IN[HDFS Input] --> B1[Block 1 to Map Task 1]
    HDFS_IN --> B2[Block 2 to Map Task 2]
    HDFS_IN --> B3[Block 3 to Map Task 3]
    HDFS_IN --> B4[Block 4 to Map Task 4]

    B1 --> S0A[Partition 0]
    B1 --> S1A[Partition 1]
    B2 --> S0B[Partition 0]
    B2 --> S1B[Partition 1]
    B3 --> S0C[Partition 0]
    B3 --> S1C[Partition 1]
    B4 --> S0D[Partition 0]
    B4 --> S1D[Partition 1]

    S0A --> R0[Reducer 0]
    S0B --> R0
    S0C --> R0
    S0D --> R0

    S1A --> R1[Reducer 1]
    S1B --> R1
    S1C --> R1
    S1D --> R1

    R0 --> OUT0[part-r-00000]
    R1 --> OUT1[part-r-00001]

    OUT0 --> HDFS_OUT[HDFS Output]
    OUT1 --> HDFS_OUT

    style HDFS_IN fill:#1d3557,color:#fff,stroke:#1d3557
    style R0 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style R1 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style HDFS_OUT fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

---

## 4. Fault Tolerance - MapReduce's Superpower

```mermaid
flowchart TD
    FAULT[Failure detected by ApplicationMaster] --> TYPE{What failed?}

    TYPE --> MAP_FAIL[Map task fails]
    MAP_FAIL --> MAP1[Reschedule on different node]
    MAP1 --> MAP2[Idempotent - rerun produces same output]
    MAP2 --> MAP3[If same task fails 4 times - job fails]

    TYPE --> RED_FAIL[Reduce task fails]
    RED_FAIL --> RED1[Reschedule on different node]
    RED1 --> RED2[Re-fetch map partitions and reprocess]
    RED2 --> RED3[Failed output discarded - never in HDFS]

    TYPE --> NODE_FAIL[Map node fails after task completed]
    NODE_FAIL --> NODE1[Map task reruns to regenerate output]
    NODE1 --> NODE2[Local storage means re-execution needed]

    TYPE --> STRAG[Straggler - task much slower than peers]
    STRAG --> SPEC[Launch speculative duplicate elsewhere]
    SPEC --> SPEC2[First to finish commits, other is killed]

    style FAULT fill:#c1121f,color:#fff,stroke:#c1121f
    style MAP_FAIL fill:#e07c24,color:#fff,stroke:#e07c24
    style RED_FAIL fill:#e07c24,color:#fff,stroke:#e07c24
    style NODE_FAIL fill:#e07c24,color:#fff,stroke:#e07c24
    style STRAG fill:#e07c24,color:#fff,stroke:#e07c24
    style SPEC2 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

**Speculative execution math:**

$$\text{Job with 1000 map tasks, 999 finish in 1 min, 1 straggler takes 10 min}$$
$$\text{Without speculation: total time} = 10 \text{ min}$$
$$\text{With speculation: duplicate launched, finishes in} \approx 1 \text{ min}$$
$$\text{Speedup} \approx 10\times \text{ at the cost of one extra task}$$

**Why Map task idempotency is guaranteed:**
- Input read from HDFS - immutable, always available
- Output written to local disk - isolated per task, no shared state
- Same input always produces same output - pure function

---

## 5. Optimization - Practical Tuning

### Choosing the Number of Reducers

```mermaid
flowchart LR
    FEW[Too few reducers] --> F1[Each reducer processes huge data]
    FEW --> F2[Long serial reduce tasks]
    FEW --> F3[Few large output files]

    GOOD[Right number] --> G1[Each reducer processes 1-5 GB]
    GOOD --> G2[Balanced parallel execution]
    GOOD --> G3[Good output file size]

    MANY[Too many reducers] --> M1[Each reducer processes tiny data]
    MANY --> M2[Thousands of tiny output files]
    MANY --> M3[Overhead of task management]

    style FEW fill:#c1121f,color:#fff,stroke:#c1121f
    style GOOD fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style MANY fill:#e07c24,color:#fff,stroke:#e07c24
```

**Rule of thumb:**

$$\text{Number of reducers} = \frac{\text{total intermediate data size}}{1 \text{ GB to } 5 \text{ GB per reducer}}$$

$$\text{Example: 100 GB intermediate data} \Rightarrow \text{use 20 to 100 reducers}$$

```java
// Set number of reducers
job.setNumReduceTasks(50);

// Map-only job (no reducers): output only needs filtering or transformation
job.setNumReduceTasks(0);  // eliminates shuffle entirely - much faster
```

---

### Compression for Shuffle Optimization

```xml
<!-- Enable Snappy compression for map output - reduces shuffle volume -->
<property>
    <name>mapreduce.map.output.compress</name>
    <value>true</value>
</property>
<property>
    <name>mapreduce.map.output.compress.codec</name>
    <value>org.apache.hadoop.io.compress.SnappyCodec</value>
</property>
```

| Codec | Speed | Ratio | Best For |
|-------|-------|-------|---------|
| Snappy | Very fast | Moderate (30-50% reduction) | Map output - CPU cost offset by I/O savings |
| LZO | Fast | Good | Map output on older clusters |
| Gzip | Slow | High (70-80% reduction) | Final output - written once, read many times |
| Zstd | Fast | High | Modern clusters - best of both |

---

### Data Skew - The Silent Performance Killer

```mermaid
flowchart LR
    SKEWED[Skewed key distribution] --> R0_HEAVY[Reducer 0 - processes 80 percent of data]
    SKEWED --> R1_IDLE[Reducers 1-99 each process 0.2 percent]
    R0_HEAVY --> DELAY[Job waits for Reducer 0 to finish]
    R1_IDLE --> WASTE[99 reducers sit idle]

    style SKEWED fill:#c1121f,color:#fff,stroke:#c1121f
    style R0_HEAVY fill:#c1121f,color:#fff,stroke:#c1121f
    style R1_IDLE fill:#e07c24,color:#fff,stroke:#e07c24
    style DELAY fill:#c1121f,color:#fff,stroke:#c1121f
    style WASTE fill:#e07c24,color:#fff,stroke:#e07c24
```

**Solution: Salting** - append a random suffix to hot keys to distribute them across multiple reducers.

```java
// Mapper: salt hot keys
public void map(LongWritable key, Text value, Context context) {
    String word = value.toString().trim();
    if (isHotKey(word)) {
        // Distribute "the" across 10 reducers as "the_0" through "the_9"
        String saltedKey = word + "_" + random.nextInt(10);
        context.write(new Text(saltedKey), one);
    } else {
        context.write(new Text(word), one);
    }
}
// Pass 1 output: ("the_0", 150000), ("the_1", 180000), ..., ("the_9", 120000)
// Pass 2 (merge): ("the", 1450000)
```

**Salting trade-off:**

$$\text{Before salting: 1 reducer processes } N \text{ records for hot key}$$
$$\text{After salting: 10 reducers each process } N/10 \text{ records}$$
$$\text{Speedup} \approx 10\times \text{ at cost of one additional MapReduce pass}$$

---

### Secondary Sort Pattern

MapReduce sorts by key before reduce. Within a key, values arrive in arbitrary order. To get ordered values within a key, use a composite key.

```java
// Composite key: (word, timestamp) - sorted by both
// But grouped by: word only
// Result: reducer receives all records for "word" sorted by timestamp

job.setSortComparatorClass(CompositeKeyComparator.class);
job.setGroupingComparatorClass(NaturalKeyGroupingComparator.class);
```

```mermaid
flowchart LR
    CKEY[Composite key - word plus timestamp] --> SORT2[Sort comparator - word then timestamp]
    SORT2 --> GROUP[Grouping comparator - group by word only]
    GROUP --> RESULT[Reducer gets values sorted by timestamp]

    style CKEY fill:#1d3557,color:#fff,stroke:#1d3557
    style RESULT fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

Elegant but complex to implement. One example of how MapReduce's simplicity forces elaborate workarounds for operations that more expressive frameworks handle naturally.

---

## 6. Multi-Step MapReduce - Chaining Jobs

Complex computations require multiple MapReduce passes. Output of one job becomes input of the next.

**Example: average word length by first letter**

```java
// Job 1: map word to (first_letter, length), reduce to (letter, sum+count)
Job job1 = Job.getInstance(conf, "Sum and Count");
FileInputFormat.addInputPath(job1,  new Path("/input/"));
FileOutputFormat.setOutputPath(job1, new Path("/tmp/intermediate/"));
job1.waitForCompletion(true);

// Job 2: map (letter, sum+count) to (letter, avg_length)
Job job2 = Job.getInstance(conf, "Compute Average");
FileInputFormat.addInputPath(job2,  new Path("/tmp/intermediate/"));
FileOutputFormat.setOutputPath(job2, new Path("/output/"));
job2.waitForCompletion(true);
```

**The critical problem:**

```mermaid
flowchart LR
    STEP1[Job 1 reads from HDFS] --> MR1[MapReduce Job 1]
    MR1 --> HDFS1[Write intermediate to HDFS - disk IO]
    HDFS1 --> STEP2[Job 2 reads from HDFS]
    STEP2 --> MR2[MapReduce Job 2]
    MR2 --> HDFS2[Write intermediate to HDFS - disk IO]
    HDFS2 --> STEP3[Job 3 reads from HDFS]
    STEP3 --> MRN[Job N]
    MRN --> HDFS_OUT2[Final output to HDFS]

    style HDFS1 fill:#c1121f,color:#fff,stroke:#c1121f
    style HDFS2 fill:#c1121f,color:#fff,stroke:#c1121f
    style MR1 fill:#457b9d,color:#fff,stroke:#457b9d
    style MR2 fill:#457b9d,color:#fff,stroke:#457b9d
```

For an N-step job:

$$\text{HDFS reads} = N, \quad \text{HDFS writes} = N, \quad \text{Total HDFS touches} = 2N$$

For a 10-step machine learning pipeline: data touches HDFS 20 times. Each touch involves disk I/O and potentially network I/O. For iterative algorithms converging over many passes, this is catastrophically slow.

**This HDFS materialization of every intermediate result is the fundamental performance limitation of MapReduce. It is the primary reason Apache Spark was invented.**

Spark keeps intermediate results in memory, eliminating the HDFS read-write cycle between steps. An iterative algorithm requiring 100 MapReduce passes (200 HDFS touches) can run in Spark with 1 HDFS read and 1 HDFS write, storing all 100 iterations in cluster RAM.

---

## 7. MapReduce in 2026 - Legacy and Relevance

### Where MapReduce Is Still Used

```mermaid
flowchart LR
    STILL[MapReduce still used for] --> S1[Very long-running batch jobs]
    STILL --> S2[Memory-constrained environments]
    STILL --> S3[Existing stable ETL pipelines]

    S1 --> S1D[12-hour jobs - fine-grained fault recovery]
    S2 --> S2D[Datasets larger than cluster RAM]
    S3 --> S3D[No benefit rewriting stable overnight jobs]

    style STILL fill:#457b9d,color:#fff,stroke:#457b9d
```

### Where MapReduce Has Been Replaced

| Workload | Why Spark Is Better |
|---------|-------------------|
| Interactive analytics | Spark runs in seconds vs minutes - in-memory |
| Machine learning | Iterative algorithms need memory, not disk per step |
| Graph processing | Iterative convergence algorithms - Spark GraphX |
| Streaming | Spark Structured Streaming vs no native MR streaming |
| SQL queries | Spark SQL or Hive on Tez vs slow MR execution |

### The Conceptual Legacy

```mermaid
flowchart TD
    MR_CONCEPTS[MapReduce conceptual contributions] --> C1[Map-shuffle-reduce pattern]
    MR_CONCEPTS --> C2[Data locality - compute to data]
    MR_CONCEPTS --> C3[Fine-grained fault-tolerant task execution]
    MR_CONCEPTS --> C4[Key-value universal data model]
    MR_CONCEPTS --> C5[Speculative execution for stragglers]

    C1 --> SPARK[Apache Spark]
    C2 --> SPARK
    C3 --> SPARK
    C4 --> SPARK
    C5 --> SPARK
    C1 --> FLINK[Apache Flink]
    C1 --> KAFKA[Kafka Streams]

    style MR_CONCEPTS fill:#1d3557,color:#fff,stroke:#1d3557
    style SPARK fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style FLINK fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style KAFKA fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

---

## Lecture Summary

```mermaid
flowchart LR
    W7L1[Week 7 Lecture 1] --> ORIGIN[Origins]
    W7L1 --> MODEL[Data Model]
    W7L1 --> EXEC[Execution]
    W7L1 --> FAULT[Fault Tolerance]
    W7L1 --> OPT[Optimization]

    ORIGIN --> O1[Functional map and reduce at cluster scale]
    ORIGIN --> O2[Stateless operations enable parallelism]

    MODEL --> M1[Universal key-value pairs]
    MODEL --> M2[Input, intermediate, and output all KV]

    EXEC --> E1[Map - read, transform, partition, spill]
    EXEC --> E2[Combiner - local pre-aggregation]
    EXEC --> E3[Shuffle - network transfer of partitions]
    EXEC --> E4[Reduce - group, aggregate, write to HDFS]

    FAULT --> F1[Idempotent map re-execution]
    FAULT --> F2[Speculative execution for stragglers]

    OPT --> OP1[Reducer count - 1-5 GB per reducer]
    OPT --> OP2[Snappy compression cuts shuffle volume]
    OPT --> OP3[Salting fixes data skew]
    OPT --> OP4[HDFS materialization - why Spark exists]

    style W7L1 fill:#1d3557,color:#fff,stroke:#1d3557
    style ORIGIN fill:#457b9d,color:#fff,stroke:#457b9d
    style MODEL fill:#457b9d,color:#fff,stroke:#457b9d
    style EXEC fill:#457b9d,color:#fff,stroke:#457b9d
    style FAULT fill:#457b9d,color:#fff,stroke:#457b9d
    style OPT fill:#457b9d,color:#fff,stroke:#457b9d
```

**Next class:** Apache Spark - architecture, how it eliminates MapReduce's limitations, RDDs and the DAG execution model, transformations vs actions, and PySpark hands-on code.

---

*BDA Spring 2026 | Week 7, Lecture 1 | The MapReduce Programming Model - Design, Execution and Optimization*
