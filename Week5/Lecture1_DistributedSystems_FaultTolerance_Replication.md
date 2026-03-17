# Big Data Analytics (BDA Spring 2026)
## Week 5, Lecture 1: Distributed System Architectures, Fault Tolerance and Replication

> In a distributed system, failure is not an exceptional event. Failure is the normal operating condition. Google's clusters of 1,000 machines see approximately one machine fail every day. The question is never whether failures will happen. The question is whether your system continues to function correctly when they do.

---

## Table of Contents

1. [What Is a Distributed System](#1-what-is-a-distributed-system)
2. [Distributed System Architectures](#2-distributed-system-architectures)
3. [Fault Tolerance - Building Systems That Survive Failure](#3-fault-tolerance---building-systems-that-survive-failure)
4. [Replication Strategies and Topologies](#4-replication-strategies-and-topologies)
5. [Failure Detection and Recovery](#5-failure-detection-and-recovery)
6. [Scalability - Growing the System](#6-scalability---growing-the-system)
7. [Complete Architecture Example](#7-complete-architecture-example)

---

## 1. What Is a Distributed System

> **A distributed system is a collection of independent computers that appears to its users as a single coherent system.**

Two things in that definition matter:

- **Independent computers** - each machine has its own processor, memory, storage, and OS. They communicate exclusively by passing messages over a network.
- **Appears as a single coherent system** - from the user's perspective they are interacting with one system. The distribution is transparent.

The gap between the physical reality (many machines, unreliable network) and the desired abstraction (one reliable system) is where all the complexity lives.

---

### The Eight Fallacies of Distributed Computing

Peter Deutsch and James Gosling at Sun Microsystems identified eight assumptions that are all wrong:

```mermaid
flowchart TD
    F[Eight Fallacies] --> F1[The network is reliable]
    F[Eight Fallacies] --> F2[Latency is zero]
    F[Eight Fallacies] --> F3[Bandwidth is infinite]
    F[Eight Fallacies] --> F4[The network is secure]
    F[Eight Fallacies] --> F5[Topology does not change]
    F[Eight Fallacies] --> F6[There is one administrator]
    F[Eight Fallacies] --> F7[Transport cost is zero]
    F[Eight Fallacies] --> F8[The network is homogeneous]

    style F fill:#c1121f,color:#fff,stroke:#c1121f
    style F1 fill:#e07c24,color:#fff,stroke:#e07c24
    style F2 fill:#e07c24,color:#fff,stroke:#e07c24
    style F3 fill:#e07c24,color:#fff,stroke:#e07c24
    style F4 fill:#e07c24,color:#fff,stroke:#e07c24
    style F5 fill:#e07c24,color:#fff,stroke:#e07c24
    style F6 fill:#e07c24,color:#fff,stroke:#e07c24
    style F7 fill:#e07c24,color:#fff,stroke:#e07c24
    style F8 fill:#e07c24,color:#fff,stroke:#e07c24
```

| Fallacy | Reality |
|---------|---------|
| Network is reliable | Packets are dropped; connections time out; routers malfunction |
| Latency is zero | Within a data center: microseconds to ms; across continents: tens to hundreds of ms |
| Bandwidth is infinite | Networks have finite capacity; shuffling large datasets saturates links |
| Network is secure | Without explicit encryption and auth, communications can be intercepted |
| Topology does not change | Servers added and removed; routes change; IPs change |
| One administrator | Large systems span multiple teams, organizations, and data centers |
| Transport cost is zero | Serialization, transmission, and deserialization all consume CPU and time |
| Network is homogeneous | Different hardware, OS, protocols, and performance throughout |

Every fallacy assumed true leads to bugs that appear only in production under load or failure.

---

## 2. Distributed System Architectures

### Architecture 1 - Client-Server

```mermaid
flowchart LR
    C1[Client] --> S[Server - stateful]
    C2[Client] --> S
    C3[Client] --> S
    S --> DB[State and data]

    style S fill:#1d3557,color:#fff,stroke:#1d3557
    style DB fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style C1 fill:#457b9d,color:#fff,stroke:#457b9d
    style C2 fill:#457b9d,color:#fff,stroke:#457b9d
    style C3 fill:#457b9d,color:#fff,stroke:#457b9d
```

| Aspect | Detail |
|--------|--------|
| Strengths | Simple to understand; clear separation of concerns; easy to reason about state |
| Weaknesses | Server is a single point of failure and a scalability bottleneck |
| How to scale | Vertical scaling, read replicas, load balancers across multiple instances |
| Examples | Traditional web apps, REST APIs, most database clients |

---

### Architecture 2 - Master-Worker

```mermaid
flowchart TD
    CLIENT[Client request] --> MASTER[Master node]
    MASTER --> DIVIDE[Divide work into tasks]
    DIVIDE --> W1[Worker 1]
    DIVIDE --> W2[Worker 2]
    DIVIDE --> W3[Worker 3]
    W1 -->|Result| MASTER
    W2 -->|Result| MASTER
    W3 -->|Result| MASTER
    MASTER --> AGG[Aggregate and return result]
    W1 -->|Heartbeat| MASTER
    W2 -->|Heartbeat| MASTER
    W3 -->|Heartbeat| MASTER

    style MASTER fill:#1d3557,color:#fff,stroke:#1d3557
    style W1 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style W2 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style W3 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style AGG fill:#457b9d,color:#fff,stroke:#457b9d
```

The master receives requests, divides work, assigns to workers, tracks completion, collects results, and handles worker failures by reassigning tasks.

| Aspect | Detail |
|--------|--------|
| Strengths | Clear responsibility boundaries; workers are simple and stateless; master has global scheduling view |
| Weakness | Master is a single point of failure and coordination bottleneck |
| Master replication | Standby master takes over on primary failure; Hadoop Active and Standby NameNode |
| Master sharding | Divide responsibilities across multiple masters, each owning a subset |
| Examples | Hadoop MapReduce, Apache Spark, HDFS |

> Master replication is a consensus problem. Which replica is the current master? HDFS uses ZooKeeper (ZAB protocol, a Paxos variant) to elect the active NameNode and ensure only one master is active. This is exactly what Raft's election mechanism solves.

---

### Architecture 3 - Peer-to-Peer

```mermaid
flowchart LR
    N1[Node 1] <--> N2[Node 2]
    N2 <--> N3[Node 3]
    N3 <--> N4[Node 4]
    N4 <--> N1
    N1 <--> N3
    N2 <--> N4

    style N1 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style N2 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style N3 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style N4 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

No master. All nodes are equal - every node can act as both client and server.

| Mechanism | Purpose |
|-----------|---------|
| Consistent hashing | Each node owns a key range on a hash ring; any node finds the owner of any key without central coordination |
| Gossip protocols | Nodes periodically exchange state with random neighbors; information spreads exponentially fast without a master |
| Quorum reads and writes | Operations require acknowledgment from a configurable number of replicas; tunable consistency |

| Aspect | Detail |
|--------|--------|
| Strengths | No single point of failure; scales naturally; no coordination bottleneck |
| Weaknesses | Coordination is complex; consistency harder to achieve without a master |
| Examples | BitTorrent, Cassandra, DynamoDB, Bitcoin |

---

### Architecture 4 - Microservices

```mermaid
flowchart LR
    GW[API Gateway] --> AUTH[Auth service]
    GW --> SEARCH[Search service]
    GW --> ORDER[Order service]
    GW --> PAY[Payment service]
    ORDER --> PAY
    ORDER --> INV[Inventory service]
    PAY --> NOTIF[Notification service]

    style GW fill:#1d3557,color:#fff,stroke:#1d3557
    style AUTH fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style SEARCH fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style ORDER fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style PAY fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style INV fill:#457b9d,color:#fff,stroke:#457b9d
    style NOTIF fill:#457b9d,color:#fff,stroke:#457b9d
```

Each service owns a specific business capability, its own database, and communicates via APIs or message queues.

| Strength | Weakness |
|---------|---------|
| Independent deployability - update one service without redeploying others | All network fallacies apply to every inter-service call |
| Independent scalability - scale payment service separately from search | Distributed transactions across services require saga patterns or two-phase commit |
| Technology diversity - different services can use different languages and DBs | Debugging requires tracing requests across dozens of services |
| Fault isolation - a bug in one service does not necessarily bring down others | Operational complexity grows with number of services |

Used by Netflix (600+ services), Uber, and Amazon. Data pipelines in modern organizations are typically implemented as microservices.

---

### Architecture 5 - Lambda and Kappa

```mermaid
flowchart LR
    DATA[Incoming data] --> BATCH[Batch layer - all history, high accuracy]
    DATA --> SPEED[Speed layer - real-time, low latency]
    BATCH --> QUERY[Query layer - merge both outputs]
    SPEED --> QUERY
    QUERY --> USER[User query result]

    style DATA fill:#1d3557,color:#fff,stroke:#1d3557
    style BATCH fill:#457b9d,color:#fff,stroke:#457b9d
    style SPEED fill:#e07c24,color:#fff,stroke:#e07c24
    style QUERY fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

| Architecture | Approach | Trade-off |
|-------------|---------|----------|
| Lambda | Batch layer for history plus speed layer for real-time; query merges both | High accuracy and low latency, but two codebases to maintain |
| Kappa | Stream-only; historical reprocessing by replaying the event log | Simpler architecture; only works if stream processor handles all patterns |

Full detail covered in Week 7.

---

## 3. Fault Tolerance - Building Systems That Survive Failure

### The Fundamental Mechanism - Redundancy

```mermaid
flowchart TD
    RED[Redundancy at every level] --> HW[Hardware redundancy]
    RED --> DATA[Data redundancy]
    RED --> SVC[Service redundancy]
    RED --> GEO[Geographic redundancy]

    HW --> H1[Dual power supplies, RAID, redundant NICs]
    DATA --> D1[Multiple copies on multiple machines]
    SVC --> S1[Multiple instances behind load balancer]
    GEO --> G1[Multiple data centers in different regions]

    style RED fill:#1d3557,color:#fff,stroke:#1d3557
    style HW fill:#457b9d,color:#fff,stroke:#457b9d
    style DATA fill:#457b9d,color:#fff,stroke:#457b9d
    style SVC fill:#457b9d,color:#fff,stroke:#457b9d
    style GEO fill:#457b9d,color:#fff,stroke:#457b9d
```

Each level of redundancy adds cost. The engineering challenge is determining how much redundancy is sufficient for your reliability requirements.

---

## 4. Replication Strategies and Topologies

### Synchronous Replication

```mermaid
flowchart LR
    C[Client write] --> LDR[Leader writes locally]
    LDR --> R1[Replica 1 writes and ACKs]
    LDR --> R2[Replica 2 writes and ACKs]
    R1 --> DONE[All ACKs received]
    R2 --> DONE
    DONE --> ACK[Leader responds - success]

    style C fill:#1d3557,color:#fff,stroke:#1d3557
    style LDR fill:#457b9d,color:#fff,stroke:#457b9d
    style R1 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style R2 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style ACK fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

Client does not receive success until all replicas confirm.

| Property | Value |
|---------|-------|
| Data loss on leader crash | None - replicas have all committed data |
| Write latency | Higher - must wait for slowest replica |
| Used by | PostgreSQL sync replication, Raft systems, HDFS critical writes, Spanner |

---

### Asynchronous Replication

```mermaid
flowchart LR
    C2[Client write] --> LDR2[Leader writes locally]
    LDR2 --> ACK2[Immediately respond - success]
    LDR2 -->|Background| R3[Replica 1 - catches up later]
    LDR2 -->|Background| R4[Replica 2 - catches up later]

    style C2 fill:#1d3557,color:#fff,stroke:#1d3557
    style LDR2 fill:#457b9d,color:#fff,stroke:#457b9d
    style ACK2 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style R3 fill:#e07c24,color:#fff,stroke:#e07c24
    style R4 fill:#e07c24,color:#fff,stroke:#e07c24
```

Client gets success immediately; replication happens in the background.

| Property | Value |
|---------|-------|
| Data loss risk | Yes - if leader crashes before replicas catch up, acknowledged writes may be lost |
| Write latency | Lower - client does not wait for replicas |
| Used by | MySQL async replication, Cassandra with ONE consistency, MongoDB default |

---

### Semi-Synchronous Replication

Wait for at least one replica (not all) before responding to client. Eliminates the single-machine data loss risk while avoiding waiting for all replicas including slow or distant ones. Used by MySQL semi-synchronous replication.

---

### Replication Topologies

```mermaid
flowchart TD
    TOPO[Replication Topologies] --> SL[Single-leader]
    TOPO --> ML[Multi-leader]
    TOPO --> LL[Leaderless - Dynamo style]

    SL --> SL1[One node accepts all writes]
    SL --> SL2[Replicas serve reads]
    SL --> SL3[Leader failure triggers Raft election]

    ML --> ML1[Multiple nodes accept writes]
    ML --> ML2[Used for multi-region writes]
    ML --> ML3[Requires conflict resolution]

    LL --> LL1[Writes go to multiple replicas at once]
    LL --> LL2[Consistency via quorums - W plus R above N]
    LL --> LL3[Used by Cassandra and DynamoDB]

    style TOPO fill:#1d3557,color:#fff,stroke:#1d3557
    style SL fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style ML fill:#e07c24,color:#fff,stroke:#e07c24
    style LL fill:#457b9d,color:#fff,stroke:#457b9d
```

**Multi-leader conflict resolution strategies:**

| Strategy | How It Works | Risk |
|---------|-------------|------|
| Last-write-wins | Timestamp determines winner | Clock skew can cause newer writes to lose |
| Application merge | Application-defined merge function | Requires custom logic per data type |
| CRDTs | Data structures that merge automatically without conflicts | Complex to design; not suitable for all data types |

**Leaderless consistency guarantee:** if W replicas must confirm a write and R replicas must respond to a read, then W + R > N ensures the read quorum always overlaps the write quorum.

**Read repair mechanisms in leaderless systems:**

| Mechanism | How It Works |
|-----------|-------------|
| Version vectors | Each write carries a version number; read returns all versions and resolves conflicts |
| Last-write-wins | Timestamp determines most recent; simple but clock-skew risk |
| Read repair | Read that detects stale replicas immediately updates them to latest |
| Anti-entropy | Background process continuously compares and repairs replica differences |

---

## 5. Failure Detection and Recovery

### Heartbeat-Based Detection

```mermaid
flowchart LR
    N[Node sends heartbeat every T seconds] --> MON[Monitor or peer node]
    MON --> CHECK{Heartbeat received within timeout?}
    CHECK -->|Yes| ALIVE[Node marked healthy]
    CHECK -->|No| DEAD[Node presumed failed]
    DEAD --> RECOVER[Trigger recovery process]

    style N fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style CHECK fill:#1d3557,color:#fff,stroke:#1d3557
    style ALIVE fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style DEAD fill:#c1121f,color:#fff,stroke:#c1121f
```

**The timeout trade-off:**

| Timeout Setting | Problem |
|----------------|---------|
| Too short | False positives - slow nodes declared dead; split-brain if old leader recovers after new election |
| Too long | Slow failure detection - applications experience full timeout duration as downtime |

**Phi Accrual Failure Detector** (used by Cassandra and Akka): instead of a binary alive/dead decision, continuously calculates a suspicion level phi based on the statistical distribution of heartbeat arrival times. As heartbeats become increasingly late relative to historical patterns, phi rises. Applications set a threshold phi at which they declare failure. Adapts to changing network conditions and reduces false positives.

---

### Recovery by Failure Type

```mermaid
flowchart TD
    FAIL[Failure detected] --> TYPE{What failed?}
    TYPE --> WRK[Worker node in master-worker]
    TYPE --> LDR[Leader in single-leader replication]
    TYPE --> NET[Network partition]

    WRK --> W1[Master marks worker dead]
    W1 --> W2[Reassign all tasks to healthy workers]
    W2 --> W3[Restart tasks - must be idempotent]

    LDR --> L1[Followers detect missed heartbeats]
    L1 --> L2[Raft election begins]
    L2 --> L3[Most up-to-date follower wins]
    L3 --> L4[Clients find new leader via etcd]

    NET --> N1[CP systems stop serving to stay consistent]
    NET --> N2[AP systems keep serving then reconcile]

    style FAIL fill:#c1121f,color:#fff,stroke:#c1121f
    style WRK fill:#e07c24,color:#fff,stroke:#e07c24
    style LDR fill:#e07c24,color:#fff,stroke:#e07c24
    style NET fill:#e07c24,color:#fff,stroke:#e07c24
    style W3 fill:#457b9d,color:#fff,stroke:#457b9d
    style L4 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

Tasks in Hadoop MapReduce are designed to be **idempotent**: running the same task twice produces the same result. This makes it safe to restart tasks that may have partially completed on a failed worker.

---

## 6. Scalability - Growing the System

### Vertical vs Horizontal Scaling

```mermaid
flowchart LR
    subgraph VERT[Vertical Scaling - Scale Up]
        VS1[Replace server with bigger server]
        VS2[More CPUs, RAM, faster storage]
        VS3[Hard upper limit exists]
        VS4[Disproportionately expensive]
        VS5[Still a single point of failure]
    end
    subgraph HORIZ[Horizontal Scaling - Scale Out]
        HS1[Add more machines to cluster]
        HS2[Linear cost scaling]
        HS3[No hard upper limit]
        HS4[Natural fault tolerance]
        HS5[Requires distributed software]
    end

    style VERT fill:#e07c24,color:#fff,stroke:#e07c24
    style HORIZ fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

All Big Data frameworks (Hadoop, Spark, Kafka, Cassandra) are designed for horizontal scalability on commodity clusters.

**Amdahl's Law revisited:** the serial fraction of your computation limits horizontal scaling. A workload that is 95% parallelizable achieves at most 20x speedup regardless of cluster size. Designing for horizontal scalability means minimizing the serial fraction: the parts requiring coordination, synchronization, or central processing.

---

### Replication Factor Trade-offs

```mermaid
xychart-beta
    title "Availability vs Coordination Overhead by Replication Factor"
    x-axis ["RF1", "RF2", "RF3", "RF4", "RF5", "RF6"]
    y-axis "Relative Value" 0 --> 10
    bar [1, 5, 8, 9, 9.5, 9.6]
```

| Replication Factor | Tolerates | Write Overhead | Typical Use |
|-------------------|----------|---------------|-------------|
| 1 | Nothing | Minimal | Development only |
| 2 | One failure | Low | Non-critical data |
| 3 | One failure | Moderate | Standard production |
| 5 | Two simultaneous failures | Higher | Multi-region deployments |
| 6+ | Diminishing returns | High | Rarely justified |

3 to 5 replicas is the production sweet spot. Beyond 5, the consistency and coordination overhead exceeds the availability benefit.

---

## 7. Complete Architecture Example

A global ride-sharing platform requires multiple consistency models within one application.

```mermaid
flowchart TD
    USER[User request] --> GW2[API Gateway]
    GW2 --> LOC[Driver location service]
    GW2 --> MATCH[Matching service]
    GW2 --> PYMNT[Payment service]
    GW2 --> HIST[Trip history service]

    LOC --> CASS[Cassandra - AP - peer to peer]
    MATCH --> MW[Master-worker - Raft elected master]
    PYMNT --> CRDB[CockroachDB - CP - sync replication]
    HIST --> LAMBDA[Lambda architecture]

    LAMBDA --> KAFKA[Kafka speed layer]
    LAMBDA --> SPARK[Spark batch layer]

    style USER fill:#1d3557,color:#fff,stroke:#1d3557
    style GW2 fill:#1d3557,color:#fff,stroke:#1d3557
    style CASS fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style MW fill:#457b9d,color:#fff,stroke:#457b9d
    style CRDB fill:#c1121f,color:#fff,stroke:#c1121f
    style LAMBDA fill:#e07c24,color:#fff,stroke:#e07c24
```

| Component | Architecture | Consistency | Reasoning |
|-----------|-------------|-------------|-----------|
| Driver location | Cassandra, RF 3 per region, quorum reads and writes | AP - eventual | 4 Hz writes from millions of drivers; 500ms stale location is fine for matching |
| Matching service | Master-worker; Raft-elected master via ZooKeeper | Strong coordination | Computationally intensive; master partitions geographic area into cells |
| Payment service | CockroachDB; synchronous replication; serializable | CP - strong | Double-charging is unacceptable; brief unavailability during partition is acceptable |
| Trip history | Lambda; Kafka speed layer plus Spark batch; Parquet on S3 | Eventual for feeds, consistent for reports | Low-latency recent queries plus high-accuracy batch analytics |

One platform, four different distributed system patterns, because each component has different requirements. This is real distributed systems architecture.

---

## Lecture Summary

```mermaid
flowchart LR
    W5L1[Week 5 Lecture 1] --> DEF[Distributed Systems]
    W5L1 --> ARCH[Architectures]
    W5L1 --> FT[Fault Tolerance]
    W5L1 --> REP[Replication]
    W5L1 --> SCALE[Scalability]

    DEF --> D1[Independent machines as one system]
    DEF --> D2[Eight fallacies - design with reality]

    ARCH --> A1[Client-Server, Master-Worker, P2P]
    ARCH --> A2[Microservices, Lambda, Kappa]

    FT --> F1[Redundancy at every level]
    FT --> F2[Heartbeats and phi accrual detection]
    FT --> F3[Idempotent tasks for safe restart]

    REP --> R1[Sync - no loss, higher latency]
    REP --> R2[Async - low latency, loss risk]
    REP --> R3[Single-leader, multi-leader, leaderless]

    SCALE --> S1[Horizontal is the Big Data way]
    SCALE --> S2[Minimize serial fraction - Amdahl]
    SCALE --> S3[RF 3 to 5 is the production sweet spot]

    style W5L1 fill:#1d3557,color:#fff,stroke:#1d3557
    style DEF fill:#457b9d,color:#fff,stroke:#457b9d
    style ARCH fill:#457b9d,color:#fff,stroke:#457b9d
    style FT fill:#457b9d,color:#fff,stroke:#457b9d
    style REP fill:#457b9d,color:#fff,stroke:#457b9d
    style SCALE fill:#457b9d,color:#fff,stroke:#457b9d
```

**Next class:** Hadoop ecosystem deep dive - HDFS architecture, NameNode and DataNode relationship, block replication, and how HDFS handles node failures.

---

*BDA Spring 2026 | Week 5, Lecture 1 | Distributed System Architectures, Fault Tolerance and Replication*
