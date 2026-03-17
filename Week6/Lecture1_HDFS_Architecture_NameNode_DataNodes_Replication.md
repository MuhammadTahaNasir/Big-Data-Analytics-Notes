# Big Data Analytics (BDA Spring 2026)
## Week 6, Lecture 1: HDFS Architecture - NameNode, DataNodes, Block Replication and Fault Tolerance

> HDFS is one of the most elegant implementations of distributed systems principles ever built. When you understand HDFS deeply, you are not just learning one file system. You are seeing every distributed systems concept from Weeks 3-5 applied in a real production system that has processed more data than almost any other software in history.

---

## Table of Contents

1. [Design Philosophy of HDFS](#1-design-philosophy-of-hdfs)
2. [HDFS Architecture - The Complete Picture](#2-hdfs-architecture---the-complete-picture)
3. [Block Replication - Durability by Design](#3-block-replication---durability-by-design)
4. [HDFS Read and Write Paths](#4-hdfs-read-and-write-paths)
5. [HDFS in Practice - Commands and Configuration](#5-hdfs-in-practice---commands-and-configuration)
6. [HDFS Limitations and the Modern Perspective](#6-hdfs-limitations-and-the-modern-perspective)

---

## 1. Design Philosophy of HDFS

HDFS was created in 2004 by Doug Cutting and Mike Cafarella as an open-source implementation of the Google File System (GFS) paper published in 2003. It became the foundation of the entire Hadoop ecosystem.

### The Four Core Design Requirements

```mermaid
flowchart TD
    REQ[HDFS Design Requirements] --> R1[Commodity hardware]
    REQ --> R2[Large files, sequential access]
    REQ --> R3[Write-once, read-many]
    REQ --> R4[High throughput over low latency]

    R1 --> R1D[1000 machines expect 1 failure per day]
    R1 --> R1E[Failure must be normal, not exceptional]

    R2 --> R2D[Files range from 100 MB to terabytes]
    R2 --> R2E[Read entire file start to end]

    R3 --> R3D[Files created once, never updated in place]
    R3 --> R3E[Massively simplifies consistency]

    R4 --> R4D[10 min vs 8 min is acceptable]
    R4 --> R4E[GB per second throughput is required]

    style REQ fill:#1d3557,color:#fff,stroke:#1d3557
    style R1 fill:#c1121f,color:#fff,stroke:#c1121f
    style R2 fill:#457b9d,color:#fff,stroke:#457b9d
    style R3 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style R4 fill:#e07c24,color:#fff,stroke:#e07c24
```

Every HDFS design decision traces back to these four requirements. Every time something seems unusual, trace it back here and it will make sense.

---

## 2. HDFS Architecture - The Complete Picture

HDFS follows the **master-worker** architecture from Week 5: one NameNode master, many DataNode workers.

```mermaid
flowchart TD
    CLIENT[HDFS Client] -->|Metadata operations only| NN[NameNode - master]
    CLIENT -->|Data read and write directly| DN1[DataNode 1]
    CLIENT -->|Data read and write directly| DN2[DataNode 2]
    CLIENT -->|Data read and write directly| DN3[DataNode 3]
    DN1 -->|Heartbeat every 3s| NN
    DN2 -->|Heartbeat every 3s| NN
    DN3 -->|Heartbeat every 3s| NN
    DN1 -->|Block report every 6h| NN
    DN2 -->|Block report every 6h| NN
    DN3 -->|Block report every 6h| NN
    SNN[Secondary NameNode] -->|Checkpoint FsImage| NN

    style NN fill:#1d3557,color:#fff,stroke:#1d3557
    style CLIENT fill:#457b9d,color:#fff,stroke:#457b9d
    style DN1 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style DN2 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style DN3 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style SNN fill:#e07c24,color:#fff,stroke:#e07c24
```

**Critical architectural rule:** the NameNode is NEVER in the data transfer path. It handles only metadata. All actual bytes flow directly between clients and DataNodes.

---

### The NameNode - The Brain of HDFS

The NameNode runs on a dedicated high-memory server. It stores NO file data. It stores only the metadata - the complete map of the file system.

**Two critical in-memory data structures:**

```mermaid
flowchart LR
    NN2[NameNode RAM] --> NS[Namespace]
    NN2 --> BM[Block Map]

    NS --> NS1[Directory tree of entire filesystem]
    NS --> NS2[Every directory, file, and path]
    NS --> NS3[File to block list mapping]

    BM --> BM1[Block ID to DataNode list]
    BM --> BM2[Which 3 DataNodes hold each block]
    BM --> BM3[Rebuilt from DataNode reports on startup]

    style NN2 fill:#1d3557,color:#fff,stroke:#1d3557
    style NS fill:#457b9d,color:#fff,stroke:#457b9d
    style BM fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

**Concrete example:** a 512 MB file at `/data/weblog.txt` with default 128 MB block size:

$$\text{Number of blocks} = \left\lceil \frac{512 \text{ MB}}{128 \text{ MB}} \right\rceil = 4 \text{ blocks}$$

| Block | Bytes | Replica 1 | Replica 2 | Replica 3 |
|-------|-------|-----------|-----------|-----------|
| Block 1 | 0 - 128 MB | DataNode 7 | DataNode 23 | DataNode 41 |
| Block 2 | 128 - 256 MB | DataNode 3 | DataNode 19 | DataNode 55 |
| Block 3 | 256 - 384 MB | DataNode 12 | DataNode 8 | DataNode 30 |
| Block 4 | 384 - 512 MB | DataNode 6 | DataNode 44 | DataNode 17 |

**NameNode memory requirement formula:**

$$\text{NameNode RAM} \approx \underbrace{150 \text{ bytes}}_{\text{per block entry}} \times \text{total blocks in cluster}$$

A cluster with 100 million blocks needs approximately **15 GB of NameNode RAM**. Production NameNodes have 64 GB to 128 GB of RAM for very large clusters.

**Why block locations are NOT persisted:** DataNode membership changes constantly. Storing locations persistently would require constant updates and would become inconsistent. Instead the NameNode rebuilds the Block Map fresh from DataNode reports on every startup - always reflecting current reality.

---

### NameNode Persistent State - FsImage and EditLog

The namespace (directory tree and file-to-block mappings) must survive NameNode restarts. HDFS uses the same **WAL + checkpoint** pattern studied in Week 3.

```mermaid
flowchart LR
    MOD[File system modification] --> ELOG[Append to EditLog on disk]
    ELOG --> APPLY[Apply to in-memory namespace]

    RESTART[NameNode restart] --> FSIMG[Load FsImage - base snapshot]
    FSIMG --> REPLAY[Replay EditLog - all changes since FsImage]
    REPLAY --> READY[Namespace restored]

    SNN2[Secondary NameNode - periodically] --> PULL[Pull FsImage and EditLog]
    PULL --> MERGE[Merge into new compact FsImage]
    MERGE --> PUSH[Push new FsImage back to NameNode]
    PUSH --> TRUNC[Truncate EditLog]

    style MOD fill:#1d3557,color:#fff,stroke:#1d3557
    style ELOG fill:#e07c24,color:#fff,stroke:#e07c24
    style FSIMG fill:#457b9d,color:#fff,stroke:#457b9d
    style MERGE fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style SNN2 fill:#e07c24,color:#fff,stroke:#e07c24
```

| Component | What It Is | Analogy |
|-----------|-----------|---------|
| FsImage | Complete namespace snapshot at time T | Database full backup |
| EditLog | Sequential log of all changes since last FsImage | WAL / transaction log |
| Secondary NameNode | Periodic FsImage + EditLog merger | Checkpoint process |

> **Important:** the Secondary NameNode is NOT a hot standby. It is a checkpointing helper. It cannot take over if the NameNode fails.

---

### HDFS High Availability - Active and Standby NameNode

Introduced in Hadoop 2. Solves the original HDFS single point of failure at the NameNode.

```mermaid
flowchart LR
    ACTIVE[Active NameNode] -->|Write all edits| JN1[JournalNode 1]
    ACTIVE -->|Write all edits| JN2[JournalNode 2]
    ACTIVE -->|Write all edits| JN3[JournalNode 3]
    JN1 -->|Continuously read edits| STANDBY[Standby NameNode]
    JN2 -->|Continuously read edits| STANDBY
    JN3 -->|Continuously read edits| STANDBY
    ZK[ZooKeeper] -->|Monitors active NN health| ZKF{Active fails?}
    ZKF -->|Yes - trigger failover| STANDBY
    STANDBY -->|Becomes new Active within seconds| NEW[New Active NameNode]

    style ACTIVE fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style STANDBY fill:#457b9d,color:#fff,stroke:#457b9d
    style JN1 fill:#e07c24,color:#fff,stroke:#e07c24
    style JN2 fill:#e07c24,color:#fff,stroke:#e07c24
    style JN3 fill:#e07c24,color:#fff,stroke:#e07c24
    style ZK fill:#1d3557,color:#fff,stroke:#1d3557
    style NEW fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

JournalNodes use a **Paxos-like quorum protocol** - edits are accepted when a majority of JournalNodes acknowledge. The Standby continuously replays these edits, keeping its namespace synchronized. ZooKeeper detects Active NameNode failure and triggers failover within seconds.

This is exactly the **leader election and replication** pattern from Week 4 (Raft) and Week 5 - applied to HDFS metadata management.

---

### DataNodes - The Storage Workers

Each DataNode is a commodity server with many hard drives or SSDs. DataNodes store actual block data as plain files in the local OS filesystem (ext4 or XFS on Linux) - no special on-disk format.

```mermaid
flowchart TD
    DN[DataNode responsibilities] --> STORE[Block storage]
    DN --> SERVE[Block serving]
    DN --> HB[Heartbeats]
    DN --> BR[Block reports]
    DN --> SCAN[Block scanner]

    STORE --> ST1[Blocks stored as plain files on local disk]
    STORE --> ST2[Uses local OS filesystem - ext4 or XFS]

    SERVE --> SV1[Client opens direct TCP connection]
    SERVE --> SV2[NameNode never touches data bytes]

    HB --> HB1[Every 3 seconds by default]
    HB --> HB2[Missed for 10 min means node is dead]

    BR --> BR1[Full block inventory on startup]
    BR --> BR2[Full report every 6 hours]
    BR --> BR3[How NameNode builds the Block Map]

    SCAN --> SC1[Reads and verifies checksum on every block]
    SCAN --> SC2[Reports corrupt blocks to NameNode]

    style DN fill:#1d3557,color:#fff,stroke:#1d3557
    style STORE fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style SERVE fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style HB fill:#457b9d,color:#fff,stroke:#457b9d
    style BR fill:#457b9d,color:#fff,stroke:#457b9d
    style SCAN fill:#e07c24,color:#fff,stroke:#e07c24
```

---

### Block Size - Why 128 MB?

Traditional filesystems use 4 KB or 8 KB blocks. HDFS uses 128 MB by default.

**NameNode memory benefit:**

$$\frac{1 \text{ TB file with 4 KB blocks}}{150 \text{ bytes/block entry}} = \frac{268 \text{ million blocks} \times 150 \text{ bytes}}{1} \approx 40 \text{ GB NameNode RAM for ONE file}$$

$$\frac{1 \text{ TB file with 128 MB blocks}}{150 \text{ bytes/block entry}} = \frac{8192 \text{ blocks} \times 150 \text{ bytes}}{1} \approx 1.2 \text{ MB NameNode RAM for ONE file}$$

**The three reasons for 128 MB blocks:**

| Reason | Explanation |
|--------|-------------|
| NameNode memory | 8192 blocks vs 268 million blocks for a 1 TB file |
| TCP connection overhead | Few connections per file; time spent transferring not connecting |
| Sequential I/O optimization | 128 MB reads keep disk read-ahead full; no seeks needed |

**The trade-off - the small files problem:**

$$\text{Wasted space} = 128 \text{ MB} - \text{actual file size} \approx 128 \text{ MB for a 1 KB config file}$$

A 1 KB configuration file stored in HDFS occupies a full 128 MB block. Millions of small files overwhelm NameNode memory and perform terribly. **Solution:** combine small files into SequenceFiles or Avro containers before storing in HDFS.

---

## 3. Block Replication - Durability by Design

Default replication factor is 3. HDFS tolerates simultaneous failure of any 2 DataNodes holding a given block.

### The Replication Pipeline

HDFS avoids sending data 3 times by using a streaming pipeline.

```mermaid
flowchart LR
    CLIENT2[Client] -->|64 KB packets| DN_A[DataNode A]
    DN_A -->|Forward immediately| DN_B[DataNode B]
    DN_B -->|Forward immediately| DN_C[DataNode C]
    DN_C -->|ACK| DN_B
    DN_B -->|ACK| DN_A
    DN_A -->|ACK| CLIENT2

    style CLIENT2 fill:#1d3557,color:#fff,stroke:#1d3557
    style DN_A fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style DN_B fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style DN_C fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

**Why pipelining achieves 3x replication at ~1x time:**

```mermaid
flowchart LR
    subgraph NAIVE[Naive - 3x time]
        NC[Client] -->|128 MB| NA[DN A]
        NC -->|128 MB| NB[DN B]
        NC -->|128 MB| ND[DN C]
    end
    subgraph PIPE[Pipeline - ~1x time]
        PC[Client] -->|streams 64KB packets| PA[DN A]
        PA -->|forwards in parallel| PB[DN B]
        PB -->|forwards in parallel| PD[DN C]
    end

    style NAIVE fill:#c1121f,color:#fff,stroke:#c1121f
    style PIPE fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

Data flows through the pipeline in a **streaming fashion** - DN A does not wait for the entire 128 MB before forwarding. It forwards 64 KB packets as they arrive. All three DataNodes receive data simultaneously.

**Effective write bandwidth formula:**

$$\text{Time} \approx \frac{128 \text{ MB}}{\text{min}(\text{network bandwidth}, \text{disk write speed})} + \underbrace{2 \times \text{RTT}}_{\text{pipeline setup overhead}}$$

---

### Rack-Aware Replica Placement

```mermaid
flowchart TD
    NN3[NameNode selects 3 DataNodes] --> R1P[Replica 1 placement]
    NN3 --> R2P[Replica 2 placement]
    NN3 --> R3P[Replica 3 placement]

    R1P --> RP1[Same rack as client]
    R1P --> RP1B[Minimizes write latency]

    R2P --> RP2[Different rack from Replica 1]
    R2P --> RP2B[Survives single rack failure]

    R3P --> RP3[Same rack as Replica 2]
    R3P --> RP3B[Only one inter-rack hop needed]

    style NN3 fill:#1d3557,color:#fff,stroke:#1d3557
    style R1P fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style R2P fill:#457b9d,color:#fff,stroke:#457b9d
    style R3P fill:#457b9d,color:#fff,stroke:#457b9d
```

**Result:** Rack A holds 1 replica, Rack B holds 2 replicas.

- Rack A fails: 2 replicas survive on Rack B
- Rack B fails: 1 replica survives on Rack A
- Single DataNode fails: 2 replicas survive on other DataNodes

The NameNode knows rack topology through a configurable rack awareness script that maps DataNode IP addresses to rack identifiers.

---

### Re-Replication and Block Health Monitoring

```mermaid
flowchart TD
    DN_FAIL[DataNode fails - heartbeats stop] --> DETECT[NameNode detects after 10 min timeout]
    DETECT --> SCAN2[Scan all blocks that were on failed node]
    SCAN2 --> CHECK{Replica count below target?}
    CHECK -->|Yes - under-replicated| PRIORITY{How urgent?}
    PRIORITY -->|1 replica remaining| HIGH[Highest priority - replicate now]
    PRIORITY -->|2 of 3 replicas remaining| MED[Medium priority]
    HIGH --> REREP[Schedule re-replication on healthy DN]
    MED --> REREP

    DN_RETURN[Failed DataNode returns] --> REPORT[Reports blocks to NameNode]
    REPORT --> OVER{Over-replicated now?}
    OVER -->|Yes - e.g. 4 of 3| DELETE[Schedule excess replica deletion]

    style DN_FAIL fill:#c1121f,color:#fff,stroke:#c1121f
    style DETECT fill:#e07c24,color:#fff,stroke:#e07c24
    style HIGH fill:#c1121f,color:#fff,stroke:#c1121f
    style REREP fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style DELETE fill:#457b9d,color:#fff,stroke:#457b9d
```

Re-replication is **automatic and continuous** - no operator intervention needed. The NameNode handles it in the background.

---

### HDFS Safemode on Startup

On startup the NameNode enters **safemode** - a read-only state while it waits for DataNodes to report their blocks and the Block Map is reconstructed.

$$\text{Exit safemode when} \quad \frac{\text{blocks with} \geq \text{min replicas}}{\text{total blocks}} \geq 99.9\%$$

Only after enough replicas are confirmed available does the NameNode exit safemode and allow writes.

---

## 4. HDFS Read and Write Paths

### Complete Write Path

```mermaid
flowchart TD
    W1[Client calls FileSystem.create] --> W2[Request to NameNode - create file]
    W2 --> W3[NameNode checks permissions, creates file]
    W3 --> W4[Client requests block allocation]
    W4 --> W5[NameNode picks 3 DNs - rack aware]
    W5 --> W6[NameNode returns DN list to client]
    W6 --> W7[Client builds DN1 - DN2 - DN3 pipeline]
    W7 --> W8[Stream 128 MB in 64 KB packets]
    W8 --> W9[Each DN writes to disk and forwards]
    W9 --> W10[ACKs flow back DN3 to DN2 to DN1]
    W10 --> W11[Block complete - Client notifies NameNode]
    W11 --> W12[NameNode updates Block Map]
    W12 --> W13{More blocks?}
    W13 -->|Yes| W4
    W13 -->|No| W14[Client calls close]
    W14 --> W15[NameNode marks file as complete]

    style W1 fill:#1d3557,color:#fff,stroke:#1d3557
    style W7 fill:#457b9d,color:#fff,stroke:#457b9d
    style W10 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style W15 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

**Write lease:** the NameNode grants the client exclusive write access (write lease) when the file is created. Periodically renewed. If the client crashes without closing, the NameNode eventually revokes the lease and recovers the file.

**Write failure handling:** if a DataNode fails mid-pipeline, the failed DN is removed from the pipeline, the remaining DNs continue, and the NameNode schedules re-replication after write completes.

---

### Complete Read Path

```mermaid
flowchart TD
    R1[Client calls FileSystem.open] --> R2[Request to NameNode - open file]
    R2 --> R3[NameNode returns blocks and DN locations]
    R3 --> R4[Client selects closest DN for Block 1]
    R4 --> R5[Direct TCP connection to selected DataNode]
    R5 --> R6[DataNode streams block bytes to client]
    R6 --> R7[Client verifies checksum of received data]
    R7 --> CORRUPT{Checksum pass?}
    CORRUPT -->|Fail - corrupt data| R8[Report corrupt block, retry other replica]
    CORRUPT -->|Pass| R9[Move to next block]
    R9 --> R10{More blocks?}
    R10 -->|Yes| R4
    R10 -->|No| R11[Read complete]

    style R1 fill:#1d3557,color:#fff,stroke:#1d3557
    style R5 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style CORRUPT fill:#e07c24,color:#fff,stroke:#e07c24
    style R8 fill:#c1121f,color:#fff,stroke:#c1121f
    style R11 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

**DataNode selection - data locality:** the client prefers the replica on the same machine (if it is a DataNode itself), then same rack, then any machine. Moving computation to the data rather than data to the computation - the core of MapReduce's performance model (Week 7).

**Checksum verification:** HDFS computes checksums for every chunk written. On read, the client verifies. A checksum failure triggers retry from a different replica and marks the corrupt replica for deletion and re-replication. This is why HDFS is trusted with petabytes of critical data.

---

### NameNode vs DataNode - Role Summary

| Aspect | NameNode | DataNode |
|--------|---------|---------|
| What it stores | Namespace and Block Map (in RAM) | Actual block bytes (on disk) |
| Hardware | High RAM, no need for large storage | Many large disks, modest RAM |
| Count per cluster | 1 active (plus 1 standby in HA) | Hundreds to thousands |
| Client interaction | Metadata only - open, close, stat | Data only - read and write bytes |
| Failure impact | Entire cluster unavailable (mitigated by HA) | Only blocks on that node affected |
| Heartbeat interval | Receives heartbeats | Sends heartbeat every 3 seconds |

---

## 5. HDFS in Practice - Commands and Configuration

### HDFS Shell Commands

```bash
# List directory
hdfs dfs -ls /data/

# Create directory
hdfs dfs -mkdir /data/logs/

# Upload from local to HDFS
hdfs dfs -put localfile.csv /data/logs/

# Download from HDFS to local
hdfs dfs -get /data/logs/file.csv ./

# Display file contents
hdfs dfs -cat /data/logs/file.csv

# Delete file
hdfs dfs -rm /data/logs/file.csv

# Delete directory recursively
hdfs dfs -rm -r /data/logs/

# Show disk usage (human readable)
hdfs dfs -du -h /data/

# Check replication factor of a file
hdfs dfs -stat %r /data/logs/file.csv

# Change replication factor
hdfs dfs -setrep 5 /data/important-file.csv

# Full cluster health check
hdfs fsck /
```

`hdfs fsck` scans the entire namespace and reports under-replicated blocks, corrupt blocks, and missing blocks. Running this regularly is standard cluster maintenance.

---

### WebHDFS - REST API

Any language can access HDFS over HTTP without the Hadoop Java client.

```bash
# List directory
curl "http://namenode:9870/webhdfs/v1/data/?op=LISTSTATUS"

# Read a file
curl -L "http://namenode:9870/webhdfs/v1/data/file.txt?op=OPEN"

# Write a file (two steps: NameNode redirects, then PUT to DataNode)
curl -i -X PUT "http://namenode:9870/webhdfs/v1/data/file.txt?op=CREATE"
# Response contains redirect URL pointing to a DataNode
curl -i -X PUT -T localfile.txt "http://datanode:9864/webhdfs/v1/..."
```

The two-step write is by design: NameNode redirects the actual data transfer to a DataNode, keeping the NameNode out of the data path.

---

### Key Configuration Parameters

```xml
<!-- hdfs-site.xml -->

<!-- Block size: 128 MB default (in bytes) -->
<property>
    <name>dfs.blocksize</name>
    <value>134217728</value>
</property>

<!-- Replication factor: 3 default -->
<property>
    <name>dfs.replication</name>
    <value>3</value>
</property>

<!-- NameNode metadata directory -->
<property>
    <name>dfs.namenode.name.dir</name>
    <value>file:///data/namenode</value>
</property>

<!-- DataNode data directories - one per physical disk -->
<property>
    <name>dfs.datanode.data.dir</name>
    <value>file:///data/disk1,file:///data/disk2,file:///data/disk3</value>
</property>

<!-- NameNode heap in hadoop-env.sh - set large for big clusters -->
<!-- export HDFS_NAMENODE_OPTS="-Xmx128g" -->
```

Multiple DataNode data directories (one per disk) is standard. HDFS distributes blocks across all configured directories, utilizing all disks in parallel. A DataNode with 12 disks can read/write 12 blocks simultaneously - 12x the I/O throughput of a single disk.

$$\text{DataNode throughput} \approx \text{num disks} \times \text{per-disk throughput}$$

---

## 6. HDFS Limitations and the Modern Perspective

```mermaid
flowchart TD
    LIMITS[HDFS Limitations] --> L1[NameNode scalability ceiling]
    LIMITS --> L2[No random write support]
    LIMITS --> L3[High latency]
    LIMITS --> L4[Small files problem]

    L1 --> LD1[Single Active NN serves all metadata ops]
    L1 --> LD2[Fix - HDFS Federation multiple NameNodes]

    L2 --> LD3[Append-only - cannot update middle of file]
    L2 --> LD4[Fix - build HBase on top of HDFS]

    L3 --> LD5[NameNode round trip before first byte]
    L3 --> LD6[First byte latency - 100ms or more]

    L4 --> LD7[Millions of small files exhaust NN RAM]
    L4 --> LD8[Fix - combine into SequenceFiles or Avro]

    style LIMITS fill:#c1121f,color:#fff,stroke:#c1121f
    style L1 fill:#e07c24,color:#fff,stroke:#e07c24
    style L2 fill:#e07c24,color:#fff,stroke:#e07c24
    style L3 fill:#e07c24,color:#fff,stroke:#e07c24
    style L4 fill:#e07c24,color:#fff,stroke:#e07c24
```

### HDFS vs Modern Cloud Object Storage

```mermaid
flowchart LR
    subgraph HDFS2[HDFS]
        H1[NameNode bottleneck]
        H2[On-premises only]
        H3[Ops overhead - manage cluster]
        H4[Random write not supported]
    end
    subgraph S3[Cloud Object Storage - S3, GCS, ADLS]
        S1[No metadata bottleneck]
        S2[Virtually unlimited scale]
        S3[Fully managed - no ops]
        S4[Same write-once semantics]
    end

    style HDFS2 fill:#e07c24,color:#fff,stroke:#e07c24
    style S3 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

In 2026 cloud object storage (Amazon S3, Google Cloud Storage, Azure Data Lake Storage) has largely replaced HDFS in new deployments. Spark, Presto, and Hive now work natively with object storage. The **Lakehouse architecture** (covered in Lecture 3 this week) is built on object storage rather than HDFS.

**HDFS remains relevant because:**
- Millions of existing Hadoop clusters are still running
- On-premises deployments where cloud is not an option
- Understanding HDFS gives you a deep understanding of distributed storage that transfers directly to understanding cloud object storage, because the same principles apply

---

## Lecture Summary

```mermaid
flowchart LR
    W6L1[Week 6 Lecture 1] --> DESIGN[Design Philosophy]
    W6L1 --> ARCH[Architecture]
    W6L1 --> BLOCKS[Blocks and Replication]
    W6L1 --> PATHS[Read and Write Paths]

    DESIGN --> D1[Commodity hardware - failure is normal]
    DESIGN --> D2[Large files, sequential, write-once]

    ARCH --> A1[NameNode - namespace and block map in RAM]
    ARCH --> A2[FsImage plus EditLog for durability]
    ARCH --> A3[HA with JournalNodes and Standby NN]
    ARCH --> A4[DataNodes - store blocks, send heartbeats]

    BLOCKS --> B1[128 MB blocks - NN RAM and throughput]
    BLOCKS --> B2[Pipeline replication - 3x at 1x time]
    BLOCKS --> B3[Rack-aware - tolerates rack failure]
    BLOCKS --> B4[Auto re-replication on DataNode failure]

    PATHS --> P1[Write - pipeline with ACK chain]
    PATHS --> P2[Read - direct from DN with checksum]
    PATHS --> P3[Data locality - compute moves to data]

    style W6L1 fill:#1d3557,color:#fff,stroke:#1d3557
    style DESIGN fill:#457b9d,color:#fff,stroke:#457b9d
    style ARCH fill:#457b9d,color:#fff,stroke:#457b9d
    style BLOCKS fill:#457b9d,color:#fff,stroke:#457b9d
    style PATHS fill:#457b9d,color:#fff,stroke:#457b9d
```

**Next class:** the Hadoop ecosystem continues with MapReduce internals, YARN resource management, and how HDFS data locality integrates with MapReduce task scheduling.

---

*BDA Sprinig 2026 | Week 6, Lecture 1 | HDFS Architecture - NameNode, DataNodes, Block Replication and Fault Tolerance*
