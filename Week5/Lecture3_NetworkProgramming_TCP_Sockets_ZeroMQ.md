# Big Data Analytics (BDA Spring 2026)
## Week 5, Lecture 3: Network Programming - TCP/IP, Sockets and ZeroMQ Messaging Patterns

> Every distributed system concept we have studied - Raft leader election, Cassandra gossip, Kafka producer-consumer, Spark task assignment - ultimately reduces to processes sending bytes to each other over a network. Today we understand those building blocks from first principles.

---

## Table of Contents

1. [The Network Stack](#1-the-network-stack)
2. [TCP vs UDP - The Fundamental Transport Choice](#2-tcp-vs-udp---the-fundamental-transport-choice)
3. [Socket Programming in Java](#3-socket-programming-in-java)
4. [ZeroMQ - High-Performance Messaging](#4-zeromq---high-performance-messaging)
5. [From Sockets to Distributed Systems](#5-from-sockets-to-distributed-systems)

---

## 1. The Network Stack

### OSI Model vs TCP/IP in Practice

The OSI model defines seven layers. In practice the TCP/IP stack collapses these into four layers that matter for distributed systems programming.

```mermaid
flowchart TD
    APP[Application Layer] --> TRANS[Transport Layer]
    TRANS --> NET[Network Layer]
    NET --> PHYS[Data Link and Physical Layer]

    APP --> A1[Your code lives here]
    APP --> A2[HTTP, gRPC, Kafka, Thrift, Protobuf]
    APP --> A3[Talks to transport via sockets]

    TRANS --> T1[TCP and UDP]
    TRANS --> T2[Process to process via ports]
    TRANS --> T3[Ports are 16-bit numbers 0 to 65535]

    NET --> N1[IP - IPv4 and IPv6]
    NET --> N2[Machine to machine routing]
    NET --> N3[No delivery or ordering guarantee]

    PHYS --> P1[Ethernet, WiFi, fiber optics]
    PHYS --> P2[Actual bits over physical media]

    style APP fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style TRANS fill:#457b9d,color:#fff,stroke:#457b9d
    style NET fill:#e07c24,color:#fff,stroke:#e07c24
    style PHYS fill:#1d3557,color:#fff,stroke:#1d3557
```

**The critical insight:** as a distributed systems programmer you work at the **Application-Transport boundary** - writing data to sockets, which hands it to TCP or UDP, which handles delivery, and IP routes it to the right machine. You write bytes in. Bytes come out on the other machine. The OS and hardware handle everything in between.

### Well-Known Ports in Big Data Systems

| Port | Service |
|------|---------|
| 80 | HTTP web server |
| 5432 | PostgreSQL |
| 9092 | Apache Kafka |
| 2181 | Apache ZooKeeper |
| 9000 | HDFS NameNode |
| 8080 | Spark Web UI |
| 6379 | Redis |

---

## 2. TCP vs UDP - The Fundamental Transport Choice

### TCP - Transmission Control Protocol

TCP provides a **reliable, ordered, byte-stream** communication channel.

```mermaid
flowchart LR
    subgraph HANDSHAKE[Three-Way Handshake]
        C1[Client] -->|SYN - seq X| S1[Server]
        S1 -->|SYN-ACK - seq Y, ack X+1| C1
        C1 -->|ACK - ack Y+1| S1
        S1 --> CONN[Connection established]
    end

    style HANDSHAKE fill:#1d3557,color:#fff,stroke:#1d3557
    style CONN fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

The three-way handshake costs **one round-trip time (RTT)** before any data flows.

- Same data center RTT: ~0.1 ms
- Cross-continent RTT: ~200 ms

This is why **connection pooling** (reusing established TCP connections) is essential for performance.

**TCP Key Properties:**

| Property | Mechanism | Cost |
|----------|-----------|------|
| Reliable delivery | Receiver sends ACK; sender retransmits on timeout | Overhead per segment |
| Ordered delivery | Sequence numbers; receiver reorders if needed | Buffer memory |
| Byte-stream | No message boundaries; app must add framing | Must prefix each message with its length |
| Flow control | Receiver advertises buffer space; sender stays within it | Reduced throughput if receiver is slow |
| Congestion control | Slow start and backoff on packet loss detection | Throughput adapts to network conditions |

**Message framing formula:** since TCP is a byte stream, you must add your own message boundaries.

$$\text{Message on wire} = \underbrace{[L_0 L_1]}_{\text{2-byte length}} + \underbrace{[B_0 B_1 \ldots B_{n-1}]}_{\text{n payload bytes}}$$

`DataOutputStream.writeUTF()` does exactly this - writes a 2-byte length prefix followed by the string bytes. `DataInputStream.readUTF()` reads the length first, then reads exactly that many bytes.

---

### UDP - User Datagram Protocol

UDP provides an **unreliable, unordered, datagram** channel.

```mermaid
flowchart LR
    SEND[Sender] -->|Datagram - fire and forget| NET2[Network]
    NET2 -->|May arrive| RECV[Receiver]
    NET2 -->|May be dropped| DROP[Gone forever]
    NET2 -->|May arrive out of order| RECV

    style SEND fill:#1d3557,color:#fff,stroke:#1d3557
    style DROP fill:#c1121f,color:#fff,stroke:#c1121f
    style RECV fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

**UDP Key Properties:**

| Property | Detail |
|----------|--------|
| No connection setup | No handshake; first byte goes out immediately |
| No retransmission | Lost packets are gone; no recovery |
| Datagram boundaries preserved | Each write = one packet; each read = one packet |
| No flow or congestion control | Sender can overwhelm receiver or network |
| Supports broadcast and multicast | One packet to multiple recipients |

**Maximum UDP payload size:**

$$\text{Max UDP payload} = 65535 - 8_{\text{UDP header}} - 20_{\text{IP header}} = 65507 \text{ bytes}$$

In practice, stay under **1472 bytes** to avoid IP fragmentation on standard Ethernet (MTU 1500 bytes).

---

### TCP vs UDP - Decision Framework

```mermaid
flowchart TD
    START[Choose transport] --> Q1{Can you tolerate lost messages?}
    Q1 -->|No - loss causes correctness problem| TCP[Use TCP]
    Q1 -->|Yes - occasional loss is harmless| Q2{Do you need ordering?}
    Q2 -->|Yes| TCP
    Q2 -->|No| Q3{Broadcast or multicast needed?}
    Q3 -->|Yes| UDP[Use UDP]
    Q3 -->|No| Q4{Is lowest latency critical?}
    Q4 -->|Yes| UDP
    Q4 -->|No| TCP

    style TCP fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style UDP fill:#457b9d,color:#fff,stroke:#457b9d
    style START fill:#1d3557,color:#fff,stroke:#1d3557
```

| Use Case | Protocol | Reasoning |
|---------|---------|-----------|
| Kafka producer to broker | TCP | Cannot lose messages; each event has business value |
| Database connections | TCP | Queries and results must be complete and ordered |
| HDFS block transfers | TCP | File data must arrive complete and correct |
| Cassandra gossip protocol | UDP | Membership info is redundant; occasional loss is harmless |
| DNS lookups | UDP | Fast; client just retries if no response |
| Video streaming | UDP | Stale retransmitted frame is worse than a skipped frame |
| Monitoring metrics | UDP | Better to skip a data point than add overhead to every measure |

---

## 3. Socket Programming in Java

### What Is a Socket?

A socket is an OS abstraction representing one endpoint of a network connection, identified by:

$$\text{TCP connection} = (\underbrace{src\_IP}_{}, \underbrace{src\_port}_{}, \underbrace{dst\_IP}_{}, \underbrace{dst\_port}_{})$$

This **four-tuple** uniquely identifies every active connection on the internet. The OS uses it to route incoming packets to the correct process.

From a programmer's perspective, a socket is like a file: write bytes to it, read bytes from it, close it when done. The OS handles all TCP machinery underneath.

---

### TCP Server - Step by Step

```java
public class SocketServer {
    public static void main(String[] args) throws IOException {
        // 1. Bind to port 50555 - OS registers this port for our process
        ServerSocket ss = new ServerSocket(50555);

        // 2. Block here until a client completes the TCP handshake
        Socket s = ss.accept();
        System.out.println("Connected: " + s.getInetAddress());

        // 3. Wrap the raw byte stream with typed I/O
        DataInputStream  in  = new DataInputStream(s.getInputStream());
        DataOutputStream out = new DataOutputStream(s.getOutputStream());

        // 4. Read a length-prefixed UTF string
        System.out.println(in.readUTF());

        // 5. Send response
        out.writeUTF("Thank you from Server");

        // 6. TCP FIN - begin four-way teardown
        s.close();
    }
}
```

```mermaid
flowchart LR
    BIND[new ServerSocket 50555] --> LISTEN[OS creates listen queue]
    LISTEN --> ACCEPT[ss.accept blocks here]
    ACCEPT -->|Client connects| SOCKET[Returns new Socket]
    SOCKET --> READ[in.readUTF - read length then bytes]
    READ --> WRITE[out.writeUTF - write length then bytes]
    WRITE --> CLOSE[s.close - send TCP FIN]

    style BIND fill:#1d3557,color:#fff,stroke:#1d3557
    style ACCEPT fill:#e07c24,color:#fff,stroke:#e07c24
    style SOCKET fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style READ fill:#457b9d,color:#fff,stroke:#457b9d
    style WRITE fill:#457b9d,color:#fff,stroke:#457b9d
```

**The listen queue:** the OS buffers incoming connections that completed the handshake but have not been `accept()`-ed yet. Default size is 50. High-traffic servers use `new ServerSocket(port, backlog)` with a large backlog.

---

### TCP Client - Step by Step

```java
public class SocketClient {
    public static void main(String[] args) throws IOException {
        // Initiates three-way handshake; blocks until connected or throws
        Socket c = new Socket("localhost", 50555);

        DataOutputStream out = new DataOutputStream(c.getOutputStream());
        out.writeUTF("Hello from " + c.getLocalSocketAddress());

        DataInputStream in = new DataInputStream(c.getInputStream());
        System.out.println("Server says: " + in.readUTF());

        c.close();
    }
}
```

`localhost` resolves to `127.0.0.1` - the **loopback address**. Traffic to `127.0.0.1` never leaves the machine; it loops back through the network stack. The client is automatically assigned an **ephemeral port** (high-numbered temporary port) by the OS.

---

### Handling Multiple Clients - The Concurrency Problem

```mermaid
flowchart TD
    ITER[Iterative server - BAD] --> I1[Accept client 1]
    I1 --> I2[Handle client 1 - blocks for 10 seconds]
    I2 --> I3[Accept client 2 - waited 10 seconds!]

    CONC[Concurrent server - GOOD] --> C1[Accept client 1]
    C1 --> C2[Spawn thread for client 1]
    C1 --> C3[Accept client 2 immediately]
    C3 --> C4[Spawn thread for client 2]
    C2 -->|Parallel| DONE[Both clients handled simultaneously]
    C4 -->|Parallel| DONE

    style ITER fill:#c1121f,color:#fff,stroke:#c1121f
    style CONC fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style DONE fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

```java
// Concurrent server pattern
ServerSocket ss = new ServerSocket(50555);
ExecutorService pool = Executors.newFixedThreadPool(100);
while (true) {
    Socket s = ss.accept();
    pool.submit(() -> handleClient(s));  // thread pool, not new thread per client
}
```

| Approach | Handles | Problem |
|---------|---------|---------|
| Iterative | One client at a time | All others wait |
| Thread per client | Many clients | 10,000 clients = 10,000 threads; too much memory |
| Thread pool | Many clients efficiently | Fixed overhead; standard production pattern |
| Java NIO + Netty | Hundreds of thousands | Single thread manages many connections via epoll |

**Netty** (used by Kafka, Elasticsearch, Cassandra) is built on Java NIO and handles massive connection counts without creating a thread per connection.

---

### InetAddress - Network Identity

```java
InetAddress single   = InetAddress.getByName("nu.edu.pk");
InetAddress[] multi  = InetAddress.getAllByName("www.google.com");
InetAddress local    = InetAddress.getLocalHost();

single.getHostName();        // reverse DNS: IP -> hostname
single.getHostAddress();     // IP address as string e.g. "192.168.1.1"
single.isReachable(2000);    // ping with 2000ms timeout
```

`getAllByName` returns multiple IPs because large services use **DNS round-robin load balancing** - the same hostname resolves to many servers. Different clients get different IPs, distributing load without a central load balancer. `isReachable()` is a manual health check - the same thing monitoring systems do periodically to detect failed nodes.

---

### UDP Socket Programming

```java
// UDP Server - no accept(), just receive datagrams
DatagramSocket server = new DatagramSocket(9876);
byte[] buffer = new byte[1024];
DatagramPacket pkt = new DatagramPacket(buffer, buffer.length);
server.receive(pkt);   // blocks until a datagram arrives

// Echo back to sender - address comes FROM the packet itself
DatagramPacket reply = new DatagramPacket(
    pkt.getData(), pkt.getLength(),
    pkt.getAddress(), pkt.getPort()   // sender address extracted from packet
);
server.send(reply);
```

```java
// UDP Client
DatagramSocket client = new DatagramSocket();
byte[] msg = "Hello UDP".getBytes();
InetAddress addr = InetAddress.getByName("localhost");
DatagramPacket send = new DatagramPacket(msg, msg.length, addr, 9876);
client.send(send);   // fire and forget - no waiting for connection
```

**Key UDP differences from TCP:**

| Aspect | TCP | UDP |
|--------|-----|-----|
| Connection before send | Required (handshake) | Not required |
| Sender address | Remembered by connection | Must extract from each packet |
| Buffer overflow | OS queues; sender slows down | Extra bytes silently discarded |
| Reading partial data | Yes - read as much as available | No - all or nothing per datagram |

This is exactly the Cassandra gossip pattern: a node sends a gossip message to a random peer. If lost, no problem - gossip is periodic and redundant. The information arrives eventually through other paths.

---

## 4. ZeroMQ - High-Performance Messaging

ZeroMQ (OMQ) is a messaging library that provides high-level patterns built on top of sockets. Performance: **~8 million messages/second** at **~30 microsecond latency**.

```mermaid
flowchart LR
    RAW[Raw TCP sockets] -->|Add ZeroMQ| ZMQ[ZeroMQ sockets]
    ZMQ --> Z1[Auto-reconnect on peer disconnect]
    ZMQ --> Z2[Message patterns built in]
    ZMQ --> Z3[Transport independence]
    ZMQ --> Z4[No broker required]

    Z3 --> TR1[inproc - in-process, zero-copy]
    Z3 --> TR2[ipc - same machine, Unix socket]
    Z3 --> TR3[tcp - over network]

    style RAW fill:#c1121f,color:#fff,stroke:#c1121f
    style ZMQ fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

Switch from `inproc://` for testing to `tcp://` for production with one line change. ZeroMQ queues messages for disconnected peers and delivers them on reconnection - raw TCP requires you to implement this yourself.

---

### Pattern 1 - Request-Reply (REQ-REP)

Strictly synchronous client-server. One request, one reply, repeat.

```java
// REQ Client
ZMQ.Context ctx = ZMQ.context(1);      // 1 I/O thread handles up to 1 Gbps
ZMQ.Socket sock = ctx.socket(ZMQ.REQ);
sock.connect("tcp://localhost:5555");
sock.send("Hello", 0);
byte[] reply = sock.recv(0);
System.out.println(new String(reply));
sock.close(); ctx.term();
```

```java
// REP Server
ZMQ.Context ctx = ZMQ.context(1);
ZMQ.Socket sock = ctx.socket(ZMQ.REP);
sock.bind("tcp://*:5555");             // server binds; client connects
while (true) {
    byte[] req = sock.recv(0);
    sock.send("World", 0);
}
```

```mermaid
flowchart LR
    C[REQ Client] -->|send request| S[REP Server]
    S -->|send reply| C
    C -->|send request| S
    S -->|send reply| C

    style C fill:#457b9d,color:#fff,stroke:#457b9d
    style S fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

**Lockstep constraint:** after `send()` you must call `recv()` before sending again. Calling `send()` twice in a row returns an error. This enforces the request-reply discipline.

**Maps to distributed systems:** RPC calls. Kafka producer-broker acknowledgment, Spark task submission to executors, HDFS client-to-NameNode metadata requests all follow this pattern conceptually.

---

### Pattern 2 - Publisher-Subscriber (PUB-SUB)

One-to-many broadcast with topic filtering.

```java
// PUB Publisher
ZMQ.Context ctx = ZMQ.context(1);
ZMQ.Socket pub = ctx.socket(ZMQ.PUB);
pub.bind("tcp://*:5557");
int count = 0;
while (true) {
    pub.send(("Hello World " + count++).getBytes(), 0);
}
```

```java
// SUB Subscriber
ZMQ.Context ctx = ZMQ.context(1);
ZMQ.Socket sub = ctx.socket(ZMQ.SUB);
sub.connect("tcp://localhost:5557");
sub.subscribe("World".getBytes());   // receive only msgs starting with "World"
for (int i = 0; i < 100; i++) {
    System.out.println(new String(sub.recv(0)));
}
```

```mermaid
flowchart LR
    PUB[Publisher] -->|All messages| ZMQ2[ZeroMQ filter]
    ZMQ2 -->|Matches sub1 filter| SUB1[Subscriber 1]
    ZMQ2 -->|Matches sub2 filter| SUB2[Subscriber 2]
    ZMQ2 -->|No match - dropped| DROP2[Dropped before app]

    style PUB fill:#1d3557,color:#fff,stroke:#1d3557
    style ZMQ2 fill:#e07c24,color:#fff,stroke:#e07c24
    style SUB1 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style SUB2 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style DROP2 fill:#c1121f,color:#fff,stroke:#c1121f
```

**The slow subscriber problem:** subscribers always miss the first few messages. Connection setup takes time; the publisher starts immediately. For real-time streams (stock prices, sensor readings) this is acceptable. For audit logs or event sourcing - use **Kafka**, which persists messages so late subscribers can read from the beginning.

**Maps to distributed systems:** Kafka is conceptually ZeroMQ PUB-SUB plus persistence. ZooKeeper's watch mechanism is pub-sub where clients subscribe to changes on specific nodes.

---

### ZeroMQ vs Raw Sockets vs Kafka - Decision Framework

```mermaid
flowchart TD
    NEED[What do you need?] --> Q1{Must messages survive subscriber downtime?}
    Q1 -->|Yes| KAFKA2[Use Apache Kafka]
    Q1 -->|No| Q2{Need standard patterns without broker?}
    Q2 -->|Yes| ZMQ3[Use ZeroMQ]
    Q2 -->|No - need full protocol control| Q3{Every microsecond matters?}
    Q3 -->|Yes| RAW2[Use raw sockets]
    Q3 -->|No - need RPC between services| GRPC[Use gRPC]

    style KAFKA2 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style ZMQ3 fill:#457b9d,color:#fff,stroke:#457b9d
    style RAW2 fill:#e07c24,color:#fff,stroke:#e07c24
    style GRPC fill:#457b9d,color:#fff,stroke:#457b9d
```

| Technology | When to Use | Not When |
|-----------|-------------|----------|
| Raw TCP sockets | Building a new protocol from scratch; maximum control | You need standard patterns |
| ZeroMQ | High-performance IPC; loss acceptable; no broker wanted | You need message durability |
| Apache Kafka | Durable event streams; replay needed; loss unacceptable | You need microsecond latency |
| gRPC | Synchronous RPC between microservices; typed schemas | High-frequency fire-and-forget messaging |

---

## 5. From Sockets to Distributed Systems

Every distributed system operation you have studied in this course is bytes on a socket.

```mermaid
flowchart TD
    DS[Distributed System Operations] --> RAFT2[Raft RequestVote]
    DS --> HDFS2[HDFS DataNode heartbeat]
    DS --> KAFKA3[Kafka producer send]
    DS --> SPARK2[Spark task completion]
    DS --> ZK[ZooKeeper ZAB broadcast]
    DS --> CASS[Cassandra gossip]

    RAFT2 --> R1[TCP socket write - serialized protobuf]
    HDFS2 --> H1[UDP datagram - small periodic packet]
    KAFKA3 --> K1[TCP write - Kafka binary wire protocol]
    SPARK2 --> S1[Akka over Netty NIO TCP]
    ZK --> Z1[TCP - ZAB proposal and ACK messages]
    CASS --> C1[UDP - periodic random peer gossip]

    style DS fill:#1d3557,color:#fff,stroke:#1d3557
    style RAFT2 fill:#457b9d,color:#fff,stroke:#457b9d
    style HDFS2 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style KAFKA3 fill:#457b9d,color:#fff,stroke:#457b9d
    style SPARK2 fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    style ZK fill:#457b9d,color:#fff,stroke:#457b9d
    style CASS fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

### Reading Socket Error Messages in Production

Even if you never write socket code, you will read these errors in logs constantly:

| Error Message | What It Means | Likely Cause |
|--------------|--------------|-------------|
| `Connection refused` | No process listening on that port | Server not started; wrong port |
| `Connection timed out` | SYN sent but no SYN-ACK received | Host unreachable; firewall blocking |
| `Connection reset by peer` | Remote side sent TCP RST | Server crashed; request rejected |
| `Broken pipe` | Wrote to socket whose peer already closed | Client disconnected before response |
| `Too many open files` | Process hit OS file descriptor limit | Connection leak; not closing sockets |
| `Address already in use` | Another process on this port | Previous server instance still running |

---

## Lecture Summary

```mermaid
flowchart LR
    W5L3[Week 5 Lecture 3] --> STACK[Network Stack]
    W5L3 --> TCPUDP[TCP vs UDP]
    W5L3 --> JAVA[Java Sockets]
    W5L3 --> ZMQ4[ZeroMQ]
    W5L3 --> CONN[Connection to DS]

    STACK --> ST1[App-Transport boundary is where we work]
    STACK --> ST2[Sockets abstract TCP and UDP from code]

    TCPUDP --> TU1[TCP - reliable ordered byte stream]
    TCPUDP --> TU2[UDP - unreliable unordered datagram]
    TCPUDP --> TU3[TCP for correctness, UDP for speed]

    JAVA --> JV1[ServerSocket.accept for servers]
    JAVA --> JV2[Socket constructor for clients]
    JAVA --> JV3[Thread pool for concurrent clients]
    JAVA --> JV4[Netty NIO for massive scale]

    ZMQ4 --> ZQ1[REQ-REP for synchronous RPC]
    ZMQ4 --> ZQ2[PUB-SUB for broadcast with filter]
    ZMQ4 --> ZQ3[No persistence - use Kafka for durability]

    CONN --> CN1[Every DS message is bytes on a socket]
    CONN --> CN2[Socket errors become readable]

    style W5L3 fill:#1d3557,color:#fff,stroke:#1d3557
    style STACK fill:#457b9d,color:#fff,stroke:#457b9d
    style TCPUDP fill:#457b9d,color:#fff,stroke:#457b9d
    style JAVA fill:#457b9d,color:#fff,stroke:#457b9d
    style ZMQ4 fill:#457b9d,color:#fff,stroke:#457b9d
    style CONN fill:#457b9d,color:#fff,stroke:#457b9d
```

**Next class:** Week 6 - The Hadoop Ecosystem. HDFS architecture in depth - NameNode and DataNode relationship, block distribution and replication, fault tolerance in HDFS, and the HDFS API. Everything from master-worker architectures, replication, and fault tolerance comes together in HDFS.

---

*BDA Spring 2026 | Week 5, Lecture 3 | Network Programming - TCP/IP, Sockets and ZeroMQ*
