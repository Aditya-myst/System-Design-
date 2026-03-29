#  Design a distributed key value caching system, like Memcached or Redis.
![cache_introduction](https://github.com/user-attachments/assets/c6f6ca99-e280-4c0b-84c3-fc456ada6788)

* First part of any system design interview, coming up with the features which the system should support
* you should try to list down all the features you can think of which our system should support.

## Question 1 - Amount of data we need to cache?
Let's assume we are looking to cache on the scale of Google or Twitter. The total size of the cache would be a few TBs.

## Q2- What should be the eviction strategy?
Eviction strategy = rule to decide which data to remove when cache is full .
### Common Eviction Strategies
*  LRU (Least Recently Used)
*  LFU (Least Frequqently Used)
*  FIFO (First in First Out)
*  TTL-based (Time Expiry)

### Eviction strategy directly impacts cache hit rate and memory efficiency. I would prefer LRU for temporal locality, but in read-heavy stable workloads, LFU might perform better.

## Q3- What should be the access pattern for the given cache?
There are majorly three kinds of caching systems :
### Write through cache : 
- Write is confirmed as success only if writes to DB and the cache BOTH succeed
- #### Higher write latency (2 writes)

### Write around cache :
- Write directly goes to the DB
- Cache updated only on read
  
### Write back cache :
- write is directly done to the caching layer.
- cache then asynchronously syncs this write to the DB.
- #### Leads to really quick write latency and high write throughput.
- Risk - cache crash , data losss.

## Q4 -  What is the kind of QPS we expect for the system?
-QPS (Queries Per Second) tells us how many requests our system must handle at peak.
 It directly informs:

* Number of machines required
* Network capacity
* Cache sizing
* Database throughput

💡Without estimation → risk of:

* High latency
* Backlog queues
* Node crashes

#### Example: Single Machine Limit
Suppose 1 machine can handle 1M QPS
If incoming QPS > 1M:
* Requests queue up
* Latency spikes
* System instability → failures

This is why distributed design is critical for high-scale systems.

### Realistic System Scale
For large-scale systems like Twitter or Google:

| Metric           | Approximate Value                                     |
| ---------------- | ----------------------------------------------------- |
| QPS              | 10M+                                                  |
| Users            | 100M–500M DAU                                         |
| Request patterns | Highly read-heavy for feeds/search, occasional bursts |

### Estimations should consider peak traffic, not average. Peak can be 5–10× average QPS.

### “For a system at Twitter/Google scale, I’d estimate peak QPS around 10M. This informs horizontal scaling decisions and ensures each node handles a safe fraction of total requests to avoid latency spikes or backlog.”

## Q5 - What is the Number of Machines Required to Cache?

### Step 1: Understand the Requirements
- Cache must be **low latency**, meaning all data must fit in **RAM**
- Disk-based caching → too slow for real-time requests  
- In-memory cache → **sub-millisecond latency**

**Implication:** Each machine’s RAM limits how much data it can hold.

---

### Step 2: Machine Memory Assumptions
- Typical production cache server RAM: **72 GB – 144 GB**
- Assume **72 GB per machine** (conservative estimate)

---

### Step 3: Total Cache Data
- Suppose total cache data = **30 TB**

**Minimum machines required (RAM-based):**

30 TB / 72 GB = 426 MACHINES

💡 **Insight:**  
This is the absolute minimum. More machines may be needed to handle high QPS per node.

---

### Step 4: QPS Per Machine Consideration
- Assume each machine can safely handle **1M QPS**
- Total QPS = **10M**

{Machines required for load} = 10

So,

{Total machines} =max(memory requirement , QPS requirement)


---

### Step 5: Senior-Level Insights
- Memory alone is not sufficient
- Must account for:
  - **Replication (fault tolerance)** → doubles machine count
  - **Load distribution** → prevents latency spikes
  - **Network overhead**
  - **Sharding & consistent hashing** → require extra nodes for smooth scaling

---

### Real-World Production Setup
- Memory requirement: **~426 machines**
- QPS requirement: **~10–20 machines**
- Replication factor: **2**

Total machines = approx 850

---

### Interview-Ready Answer
> To hold 30 TB in memory using 72 GB machines, we need ~426 machines as the absolute minimum. In practice, considering high QPS per node and replication for fault tolerance, we may deploy ~850 machines. This ensures low latency, high availability, and evenly distributed load.

## Design Goals

When designing a system, it’s important to clearly define the key goals and understand the trade-offs between them. The three fundamental aspects often considered are **Latency**, **Consistency**, and **Availability**.

---

### 1. Latency

**Definition:**  
Latency refers to the time it takes for a system to respond to a request.

**Key Considerations:**
- Is the system user-facing and interactive?
- Are delays noticeable or harmful to user experience?
- Are slow responses as bad as failed requests?

**Examples:**
- **Highly latency-sensitive:**
  - Search typeahead suggestions
  - Real-time bidding systems
  - Online gaming
- **Less latency-sensitive:**
  - Batch processing systems
  - Data analytics pipelines

**Takeaway:**  
If latency is critical, prioritize fast responses—even if it means sacrificing consistency or completeness.

---

### 2. Consistency

**Definition:**  
Consistency determines whether all users see the same data at the same time.

**Types:**
- **Strong Consistency:** All reads reflect the most recent write.
- **Eventual Consistency:** Data will become consistent over time.

**Key Considerations:**
- Is stale data acceptable?
- Can temporary inconsistencies cause issues?

**Examples:**
- **Requires strong consistency:**
  - Banking systems
  - Inventory management for limited stock
- **Can tolerate eventual consistency:**
  - Social media feeds
  - Caching systems

**Takeaway:**  
Choose the level of consistency based on correctness requirements and tolerance for stale data.

---

### 3. Availability

**Definition:**  
Availability refers to the system’s ability to respond to requests at all times.

**Key Considerations:**
- Is downtime acceptable?
- What are the consequences of failure?

**Examples:**
- **High availability required:**
  - E-commerce platforms
  - Messaging systems
- **Lower availability acceptable:**
  - Internal tools
  - Offline reporting systems

**Takeaway:**  
If availability is critical, the system should always respond—even if the response is slightly outdated.

---

### Trade-offs (CAP Theorem)

In distributed systems, you often cannot fully optimize all three properties simultaneously:

- **Consistency**
- **Availability**
- **Partition Tolerance**

**CAP Theorem Insight:**
- In the presence of network partitions, you must choose between **Consistency** and **Availability**.

---

### Prioritization Strategy

When all parameters matter, prioritize based on the use case:

| Use Case                  | Priority Order                  |
|--------------------------|--------------------------------|
| Financial systems        | Consistency > Availability > Latency |
| Social media feed        | Availability > Latency > Consistency |
| Search autocomplete      | Latency > Availability > Consistency |
| E-commerce checkout      | Consistency > Availability > Latency |

---

### Final Thoughts

- Clearly define system requirements before designing.
- Understand trade-offs—optimizing one dimension often impacts others.
- Align design decisions with business and user needs.
