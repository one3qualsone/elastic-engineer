# Elasticsearch Architecture Fundamentals

## Understanding Clusters, Nodes, Shards, and Distribution

Elasticsearch's architecture is designed for scale and resilience from the ground up. Understanding how these components work together is essential for designing systems that perform well and handle failures gracefully.

---

## 🏗️ The Hierarchy: Cluster → Node → Index → Shard → Document

### The Big Picture

Elasticsearch organizes everything in a clear hierarchy:

```
Cluster (my-production-cluster)
├── Node 1 (es-node-01)
│   ├── Index A
│   │   ├── Shard 0 (primary)
│   │   └── Shard 2 (replica)
│   └── Index B
│       └── Shard 1 (primary)
├── Node 2 (es-node-02)
│   ├── Index A
│   │   ├── Shard 1 (primary)
│   │   └── Shard 0 (replica)
│   └── Index B
│       └── Shard 0 (replica)
└── Node 3 (es-node-03)
    └── Index A
        └── Shard 1 (replica)

```

**Key insight:** This hierarchy is designed so that data and computation can be distributed across multiple machines, with automatic failover when things go wrong.

---

## 🌐 Clusters: The Top-Level Organization

### What is an Elasticsearch Cluster?

A **cluster** is a collection of nodes that work together to store data and provide search capabilities.

**Key characteristics:**

- **Single logical unit:** Applications see one search engine, not individual nodes
- **Shared state:** All nodes know about all other nodes and data locations
- **Automatic coordination:** Nodes automatically distribute work and handle failures
- **Horizontal scaling:** Add capacity by adding more nodes

### Cluster-Level Responsibilities

| Function | How It Works |
| --- | --- |
| **Data distribution** | Automatically spreads data across available nodes |
| **Failure detection** | Monitors node health and handles failures |
| **Load balancing** | Distributes queries across nodes for performance |
| **Coordination** | Manages cluster-wide operations like index creation |

**Real-world analogy:** Think of a cluster like a distributed library system where multiple locations work together to serve patrons, automatically redirecting requests when one location is closed.

---

## 🖥️ Nodes: The Worker Machines

### Node Types and Roles

Each node in a cluster can have one or more roles:

| Role | Responsibilities | Resource Requirements |
| --- | --- | --- |
| **Master-eligible** | Cluster coordination, metadata management | Low CPU, moderate memory |
| **Data** | Store data, execute queries | High CPU, high memory, fast storage |
| **Ingest** | Process documents during indexing | Moderate CPU, moderate memory |
| **Coordinating** | Route requests, merge query results | Moderate CPU, moderate memory |
| **Machine Learning** | Run ML jobs and models | High CPU, high memory |

### Common Node Configuration Patterns

**Small cluster (3-5 nodes):**

```
Node 1, 2, 3: master-eligible + data + ingest + coordinating

```

- All nodes can do everything
- Simple but less optimal for specialized workloads

**Medium cluster (6-15 nodes):**

```
Node 1, 2, 3: master-eligible only (dedicated masters)
Node 4-12: data + ingest
Node 13-15: coordinating only (client nodes)

```

- Dedicated masters for stability
- Coordinating nodes for application connections

**Large cluster (15+ nodes):**

```
Node 1, 2, 3: master-eligible only
Node 4-8: ingest only
Node 9-20: data (hot tier)
Node 21-30: data (warm tier)
Node 31-35: data (cold tier)
Node 36-40: coordinating only

```

- Highly specialized roles for optimal performance
- Multiple data tiers for cost optimization

### Master Node Responsibilities

**Why master nodes matter:**

- **Cluster state management:** Track which nodes are available, where data is located
- **Index management:** Create/delete indexes, update mappings
- **Shard allocation:** Decide where to place primary and replica shards
- **Node coordination:** Handle node joins/leaves, detect failures

**The master election process:**

1. When cluster starts or master fails, eligible nodes hold an election
2. Majority of master-eligible nodes must agree on new master
3. New master takes over cluster coordination
4. If no majority exists, cluster becomes read-only (split-brain prevention)

**Think First Exercise:**

> You have a 6-node cluster with 3 master-eligible nodes. Network issues split the cluster into two groups: one with 2 nodes, one with 4 nodes. Which group can elect a master and why?
> 

**Answer:**

The group with 4 nodes can elect a master because it contains a **majority** (2 out of 3) of the master-eligible nodes.

**Why this matters:**

- The group with 2 nodes only has 1 master-eligible node (minority)
- Elasticsearch requires a majority to prevent "split-brain" scenarios
- Split-brain occurs when multiple nodes think they're the master simultaneously
- This could lead to data corruption or conflicting cluster states

**The rule:** Always configure an odd number of master-eligible nodes, and ensure they're distributed across failure domains (different racks, data centers, availability zones).

---

## 📚 Indexes: Logical Data Collections

### What is an Index?

An **index** is a logical collection of related documents, similar to a database table but optimized for search.

**Index characteristics:**

- **Schema definition:** Mappings define how documents are analyzed and stored
- **Settings configuration:** Control performance, replication, and lifecycle
- **Shard distribution:** Data is automatically split across multiple shards
- **Independent operation:** Each index can have different configurations

### Index Settings vs Mappings

| Component | Purpose | Examples |
| --- | --- | --- |
| **Settings** | Control operational behavior | Number of shards, refresh interval, analyzer definitions |
| **Mappings** | Define document structure | Field types, analyzers for text fields, index/store options |

**Example index configuration:**

```json
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 2,
    "refresh_interval": "1s"
  },
  "mappings": {
    "properties": {
      "title": {"type": "text", "analyzer": "english"},
      "publish_date": {"type": "date"},
      "category": {"type": "keyword"}
    }
  }
}

```

### Index Naming Patterns

**Static indexes:** For relatively stable data

```
products-v1
users-2024
company-knowledge-base

```

**Time-based indexes:** For time-series data

```
logs-2024.01.15
metrics-2024-01
events-daily-2024.01.15

```

**Rollover indexes:** For automatic index management

```
logs-000001 → logs-000002 → logs-000003

```

---

## 🔧 Shards: The Distribution Units

### Understanding Shards

A **shard** is the fundamental unit of distribution in Elasticsearch. Each shard is a complete, independent Lucene index.

**Why shards exist:**

- **Scalability:** Distribute data across multiple machines
- **Parallelism:** Execute queries across multiple shards simultaneously
- **Fault tolerance:** Replicate shards for high availability

### Primary vs Replica Shards

| Shard Type | Purpose | Characteristics |
| --- | --- | --- |
| **Primary** | The authoritative copy of data | All writes go here first |
| **Replica** | Copy of primary for redundancy | Used for reads and failover |

**Key rules:**

- Primary and its replicas are **never** on the same node
- If a primary fails, a replica is **automatically promoted** to primary
- Replicas can serve read requests to **improve performance**

### Shard Sizing Guidelines

**The Goldilocks problem:** Not too big, not too small, just right.

| Shard Size | Problems | Recommendations |
| --- | --- | --- |
| **Too small (< 1GB)** | Overhead dominates, many small files | Merge small shards or reduce shard count |
| **Too large (> 50GB)** | Slow recovery, memory pressure | Split large shards or increase shard count |
| **Just right (10-30GB)** | Good balance of performance and manageability | Aim for this range for most use cases |

**Shard count calculation:**

```
Total data size: 300GB
Target shard size: 20GB
Optimal primary shards: 300GB ÷ 20GB = 15 shards

```

### The Immutable Shard Reality

**Critical limitation:** You cannot change the number of primary shards after index creation.

**Why this limitation exists:**

- Documents are routed to shards using a hash of the document ID
- Changing shard count would require moving documents between shards
- This would be extremely expensive for large indexes

**Solutions:**

- **Plan ahead:** Estimate future data size and choose appropriate shard count
- **Rollover patterns:** Create new indexes when current ones get too large
- **Reindex:** Copy to new index with different shard count (expensive but possible)
- **Shrink API:** Reduce shards for read-only indexes

**Think First Exercise:**

> You create an index with 5 primary shards and 1 replica (10 total shards). Later, you want to add 2 more nodes to improve performance. Can you increase the number of primary shards to take advantage of the new nodes?
> 

**Answer:**

**No, you cannot increase the number of primary shards** after index creation.

**What you CAN do:**

- **Increase replicas:** Add more replica shards to utilize the new nodes for read performance
- **Create new indexes:** Use more shards for new time-based indexes
- **Reindex:** Copy all data to a new index with more shards (expensive operation)

**What this means:**

- The number of primary shards determines the **maximum** parallelism for writes
- Adding nodes helps with replicas (read performance) but not primary distribution
- Plan your shard count carefully based on expected data growth

**Lesson:** Elasticsearch's distribution is powerful but requires upfront planning.

---

## 🔄 How Distributed Operations Work

### Write Operations: Getting Data In

**The write path for a single document:**

1. **Client sends request** to any node (coordinating node)
2. **Coordinating node determines shard** using document ID hash
3. **Request routed to primary shard** node
4. **Primary shard processes write** (index, update, or delete)
5. **Primary forwards to replicas** in parallel
6. **All replicas acknowledge** completion
7. **Coordinating node responds** to client

**Key insights:**

- **Any node can coordinate:** Clients can connect to any cluster node
- **Primary processes first:** Ensures consistency across replicas
- **Parallel replication:** Replicas are updated simultaneously for speed
- **Write requires majority:** Operation fails if replicas are unavailable

### Read Operations: Getting Data Out

**The search path for queries:**

1. **Client sends query** to coordinating node
2. **Query distributed** to all relevant shards (primary or replica)
3. **Each shard executes query** locally and returns results
4. **Coordinating node merges results** from all shards
5. **Global ranking applied** to create final result set
6. **Response sent to client**

**Performance implications:**

- **Parallel execution:** Query runs on all shards simultaneously
- **Network overhead:** Results must be collected from multiple nodes
- **Memory usage:** Coordinating node must merge large result sets
- **Load distribution:** Replicas can serve read requests

### Shard Allocation and Rebalancing

**How Elasticsearch decides where to place shards:**

| Factor | Consideration |
| --- | --- |
| **Node capacity** | Available disk space, memory, CPU |
| **Failure domains** | Spread replicas across different zones/racks |
| **Shard types** | Never place primary and replica on same node |
| **Load balancing** | Distribute shards evenly across nodes |
| **User constraints** | Honor allocation filtering rules |

**Automatic rebalancing triggers:**

- Node joins or leaves the cluster
- Node fails or becomes unavailable
- Disk space thresholds exceeded
- Allocation rules changed

**Example rebalancing scenario:**

```
Before: Node1[P0,R1], Node2[P1,R0], Node3[offline]
Node failure: Node3 fails
After: Node1[P0,R1], Node2[P1,R0], new replicas created
New node joins: Node4 joins
Final: Node1[P0], Node2[P1], Node4[R0,R1] (rebalanced)

```

---

## 📊 Capacity Planning and Scaling Patterns

### Vertical vs Horizontal Scaling

**Vertical scaling (scale up):**

- Add more CPU, memory, or storage to existing nodes
- Limited by single-machine constraints
- Good for small to medium workloads

**Horizontal scaling (scale out):**

- Add more nodes to the cluster
- Elasticsearch's preferred scaling method
- Better fault tolerance and unlimited capacity

### Scaling Patterns by Use Case

**Search applications:**

- Start with 3-node cluster for high availability
- Scale by adding more nodes and increasing replicas
- Optimize for query performance and relevance

**Observability (high-write workloads):**

- Use hot-warm-cold architecture
- Scale by adding more hot nodes for ingestion
- Move older data to cheaper warm/cold storage

**Security (long-term retention):**

- Plan for significant data growth over time
- Use cold/frozen tiers for cost-effective long-term storage
- Implement automated data lifecycle management

### Resource Planning Guidelines

**CPU recommendations:**

- **Search:** Moderate CPU for query processing
- **Indexing:** High CPU for text analysis and document processing
- **Aggregations:** High CPU for mathematical computations

**Memory recommendations:**

- **Heap size:** 50% of available RAM, max 32GB
- **OS cache:** Remaining 50% for file system caching
- **Hot nodes:** More memory for active data
- **Cold nodes:** Less memory, more storage

**Storage recommendations:**

- **Hot tier:** Fast SSDs for active data
- **Warm tier:** Slower SSDs for less active data
- **Cold tier:** Spinning disks for archived data
- **Replication:** Factor in replica storage overhead

---

## 🛡️ Fault Tolerance and Recovery

### Types of Failures

**Node failures:**

- Hardware failure, power loss, network partition
- **Elasticsearch response:** Promote replicas to primaries, rebalance shards

**Disk failures:**

- Storage device failure, corruption, disk full
- **Elasticsearch response:** Relocate shards to healthy nodes

**Network partitions:**

- Split-brain scenarios, communication loss
- **Elasticsearch response:** Majority consensus, read-only mode for minority

### Recovery Processes

**Primary shard failure:**

1. Detect primary shard is unavailable
2. Promote replica to primary status
3. Create new replica on available node
4. Update cluster state

**Node recovery:**

1. Node rejoins cluster
2. Compare local data with cluster state
3. Sync any missing data from primaries
4. Resume normal operations

**Cluster restart:**

1. Nodes discover each other
2. Elect master node
3. Recover cluster state from metadata
4. Allocate shards to available nodes

---

## 💡 Key Takeaways

✅ **Clusters provide logical unity across multiple physical machines**

✅ **Node roles should be specialized for optimal performance at scale**

✅ **Master nodes require careful configuration to prevent split-brain**

✅ **Shard count cannot be changed after index creation - plan ahead**

✅ **Primary and replica shards enable both performance and fault tolerance**

✅ **Elasticsearch automatically handles distribution and failure recovery**

✅ **Capacity planning must consider CPU, memory, and storage for each tier**

---
