# Sharding Reality Explained

## How Distributed Search Actually Works at Scale

Sharding is often misunderstood as simply "splitting data across nodes." In reality, sharding is a complex balancing act between performance, scalability, resource utilisation, and operational complexity. This guide explains how sharding actually works in production, common misconceptions, and proven strategies for designing shard architectures that scale.

---

## 🧩 The Sharding Fundamentals

### What Sharding Actually Accomplishes

**Data distribution:**

```
Index: 1TB, 10M documents
├── Shard 0: 200GB, 2M docs (Node A)
├── Shard 1: 200GB, 2M docs (Node B)
├── Shard 2: 200GB, 2M docs (Node C)
├── Shard 3: 200GB, 2M docs (Node D)
└── Shard 4: 200GB, 2M docs (Node E)

```

**Parallel processing:**

```
Search Query → Coordinator Node
               ├── Query → Shard 0 (Node A) → Results 0
               ├── Query → Shard 1 (Node B) → Results 1
               ├── Query → Shard 2 (Node C) → Results 2
               ├── Query → Shard 3 (Node D) → Results 3
               └── Query → Shard 4 (Node E) → Results 4

Coordinator merges and ranks all results → Final response

```

### The Document Routing Reality

**Routing algorithm:**

```
Document ID: "user-12345"
Hash(user-12345) = 89347291847
Target Shard = 89347291847 % 5 = 2

Document goes to Shard 2

```

**Why this matters:**

- **Deterministic:** Same document always goes to same shard
- **Balanced:** Hash function distributes evenly (in theory)
- **Immutable:** Can't change primary shard count without reindexing
- **Searchable:** Queries check all shards to find documents

**Think First Exercise:**

> You have a user activity index with 5 shards. User "alice" generates 1000 events per day. Where do all of Alice's events go, and what are the implications?
> 

**Answer:**
All of Alice's events go to the same shard (determined by hash("alice") % 5). This creates a "hot shard" problem where one shard gets disproportionate write traffic, while others remain underutilized. This is why document ID design matters for write scalability.

---

## ⚖️ The Shard Sizing Balancing Act

### The Goldilocks Problem

**Too few shards:**

```
1TB index, 2 shards = 500GB per shard
Problems:
├── Large shards are slow to recover
├── Can't utilize all cluster nodes
├── Limited query parallelism
└── Difficult to scale horizontally

```

**Too many shards:**

```
1TB index, 100 shards = 10GB per shard
Problems:
├── High coordination overhead
├── Many small segments (inefficient)
├── Memory overhead per shard
└── Complex failure scenarios

```

**Just right:**

```
1TB index, 20 shards = 50GB per shard
Benefits:
├── Good parallelism across nodes
├── Reasonable recovery times
├── Efficient resource utilization
└── Room for growth

```

### Real-World Sizing Guidelines

**Shard size targets:**

| Use Case | Target Shard Size | Reasoning |
| --- | --- | --- |
| **Search-heavy workloads** | 10-30GB | Faster queries, better cache locality |
| **Write-heavy workloads** | 30-50GB | Larger shards handle write volume better |
| **Time-series data** | 20-50GB | Balanced for retention and query performance |
| **Analytics workloads** | 50-200GB | Larger shards for aggregation efficiency |

**The mathematics of shard planning:**

```python
def calculate_shards(data_size_gb, retention_days, growth_rate_per_day, target_shard_size_gb=30):
    """Calculate optimal primary shard count"""

    # Current data size
    current_size = data_size_gb

    # Future data size
    future_size = current_size + (retention_days * growth_rate_per_day)

    # Add 20% buffer for unexpected growth
    total_size = future_size * 1.2

    # Calculate primary shards needed
    primary_shards = max(1, int(total_size / target_shard_size_gb))

    # Ensure reasonable distribution across nodes
    # Rule: No more than 3 shards per GB of heap per node
    heap_per_node_gb = 32  # Common heap size
    max_shards_per_node = heap_per_node_gb * 3

    return {
        'primary_shards': primary_shards,
        'total_size_gb': total_size,
        'avg_shard_size_gb': total_size / primary_shards,
        'min_nodes_needed': primary_shards / max_shards_per_node
    }

# Example calculation
result = calculate_shards(
    data_size_gb=500,      # Current: 500GB
    retention_days=365,    # Keep 1 year
    growth_rate_per_day=2  # Growing 2GB/day
)
print(result)
# {'primary_shards': 41, 'total_size_gb': 1476.0, 'avg_shard_size_gb': 36.0, 'min_nodes_needed': 13.67}

```

---

## 🚀 Performance Implications of Sharding

### Query Performance Patterns

**Parallel execution benefits:**

```json
// Query across 5 shards
{
  "took": 45,  // Total query time
  "hits": {"total": {"value": 50000}},
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  }
}

// Each shard processed ~10,000 documents in parallel
// vs sequential processing of 50,000 documents

```

**Coordination overhead:**

```
Query execution breakdown:
├── 5ms: Query planning and routing
├── 35ms: Parallel shard execution (max of all shards)
├── 3ms: Result aggregation and ranking
└── 2ms: Response serialization

Total: 45ms

```

**The coordination tax:**

```python
def estimate_query_time(shard_count, docs_per_shard, complexity_factor=1.0):
    """Estimate query time based on shard count"""

    # Base processing time per document (microseconds)
    base_processing_time_us = 10

    # Shard processing time (parallel)
    shard_processing_ms = (docs_per_shard * base_processing_time_us * complexity_factor) / 1000

    # Coordination overhead (increases with shard count)
    coordination_overhead_ms = 2 + (shard_count * 0.5)

    # Result merging (increases with result set size)
    merging_time_ms = max(1, shard_count * 0.2)

    total_time_ms = coordination_overhead_ms + shard_processing_ms + merging_time_ms

    return {
        'total_time_ms': total_time_ms,
        'shard_processing_ms': shard_processing_ms,
        'coordination_overhead_ms': coordination_overhead_ms,
        'merging_time_ms': merging_time_ms
    }

# Compare different shard strategies
for shard_count in [1, 5, 10, 20, 50]:
    docs_per_shard = 1000000 // shard_count
    result = estimate_query_time(shard_count, docs_per_shard)
    print(f"Shards: {shard_count:2d}, Time: {result['total_time_ms']:6.1f}ms")

```

### Write Performance and Hot Shards

**Even distribution assumption vs reality:**

```python
# Theoretical: Perfect distribution
documents_per_shard = total_documents / shard_count

# Reality: Skewed distribution
shard_distribution = {
    'shard_0': 180000,  # Hot shard (popular user IDs)
    'shard_1': 120000,  # Normal
    'shard_2': 200000,  # Hot shard (sequential IDs)
    'shard_3': 105000,  # Normal
    'shard_4': 95000    # Cold shard
}

```

**Hot shard symptoms:**

```bash
# Monitoring shows uneven resource usage
Node A (Shard 0): CPU 85%, Memory 90%, Disk I/O 95%
Node B (Shard 1): CPU 45%, Memory 60%, Disk I/O 40%
Node C (Shard 2): CPU 80%, Memory 85%, Disk I/O 90%
Node D (Shard 3): CPU 40%, Memory 55%, Disk I/O 35%
Node E (Shard 4): CPU 30%, Memory 45%, Disk I/O 25%

```

**Solutions for hot shards:**

| Problem | Solution | Implementation |
| --- | --- | --- |
| **Sequential IDs** | Add randomness to document IDs | `uuid-prefix-{sequential}` |
| **User-based hotspots** | Use composite routing | `{user_id}-{timestamp}` |
| **Time-based patterns** | Distribute by multiple factors | Include geo or category in ID |
| **Single tenant dominance** | Custom routing for large tenants | Route big tenants to dedicated shards |

---

## 🔄 Shard Lifecycle and Management

### Index Lifecycle and Sharding

**Time-based indices:**

```
logs-2024-01-01  (5 shards, 50GB)  ← Hot tier
logs-2024-01-02  (5 shards, 52GB)  ← Hot tier
logs-2024-01-03  (5 shards, 48GB)  ← Hot tier
...
logs-2023-12-01  (5 shards, 45GB)  ← Warm tier
logs-2023-11-01  (5 shards, 43GB)  ← Cold tier
logs-2023-06-01  (DELETED)         ← Expired

```

**Shard allocation evolution:**

```python
def design_lifecycle_sharding(daily_data_gb, retention_days, node_types):
    """Design sharding strategy for time-based indices"""

    lifecycle_phases = {
        'hot': {
            'duration_days': 7,
            'target_shard_size_gb': 20,  # Smaller for frequent writes
            'node_type': 'hot',
            'replica_count': 1
        },
        'warm': {
            'duration_days': 30,
            'target_shard_size_gb': 50,  # Larger for read-only
            'node_type': 'warm',
            'replica_count': 0  # Reduce replicas to save space
        },
        'cold': {
            'duration_days': retention_days - 37,
            'target_shard_size_gb': 100,  # Largest for archive
            'node_type': 'cold',
            'replica_count': 0
        }
    }

    primary_shards = max(1, int(daily_data_gb / lifecycle_phases['hot']['target_shard_size_gb']))

    return {
        'primary_shards': primary_shards,
        'lifecycle_phases': lifecycle_phases,
        'daily_indices': True,
        'rollover_size_gb': primary_shards * lifecycle_phases['hot']['target_shard_size_gb']
    }

```

### Force Merge and Optimization

**Segment optimization over time:**

```
Hot Phase (Active writes):
├── Many small segments (frequent merges)
├── 1 primary shard = 20-50 segments
└── Background merging during low traffic

Warm Phase (Read-only):
├── Force merge to 1 segment per shard
├── Optimal for query performance
└── Reduces memory and disk usage

Cold Phase (Archive):
├── Remains at 1 segment per shard
├── Potentially moved to slower storage
└── May use searchable snapshots

```

**Force merge strategy:**

```python
def schedule_force_merge(index_pattern, max_segments=1):
    """Schedule force merge for read-only indices"""

    force_merge_config = {
        "policy": {
            "phases": {
                "warm": {
                    "actions": {
                        "forcemerge": {
                            "max_num_segments": max_segments
                        }
                    }
                }
            }
        }
    }

    # Benefits of force merge to 1 segment:
    benefits = {
        'query_performance': '+20-40%',  # Fewer segments to search
        'memory_usage': '-30-50%',       # Less segment metadata
        'disk_space': '-10-20%',         # Better compression
        'cache_efficiency': '+significant'  # Better OS page cache usage
    }

    return force_merge_config, benefits

```

---

## 🎯 Advanced Sharding Strategies

### Custom Routing for Predictable Patterns

**Tenant-based routing:**

```json
// Index document with custom routing
PUT /multi_tenant_index/_doc/doc1?routing=tenant_large_corp
{
  "tenant_id": "tenant_large_corp",
  "content": "Document content",
  "timestamp": "2024-01-15T10:00:00Z"
}

// Search within specific tenant (targets specific shards)
GET /multi_tenant_index/_search?routing=tenant_large_corp
{
  "query": {
    "bool": {
      "filter": [{"term": {"tenant_id": "tenant_large_corp"}}],
      "must": [{"match": {"content": "search terms"}}]
    }
  }
}

```

**Benefits of custom routing:**

- **Query performance:** Search hits only relevant shards
- **Data locality:** Tenant data co-located on same shards
- **Resource isolation:** Large tenants don't impact small ones
- **Operational simplicity:** Easier tenant-specific operations

**Routing key design patterns:**

| Pattern | Routing Key | Use Case | Trade-offs |
| --- | --- | --- | --- |
| **Tenant-based** | `tenant_id` | Multi-tenant SaaS | Risk of hot shards for large tenants |
| **Geographic** | `region_code` | Global applications | Uneven regional distribution |
| **Time-based** | `date_bucket` | Analytics workloads | Temporal hot spots |
| **Category-based** | `category_id` | E-commerce | Category popularity skew |
| **Hybrid** | `{tenant}_{region}` | Complex applications | More balanced distribution |

### Shard Allocation Awareness

**Rack/zone aware allocation:**

```yaml
# Elasticsearch configuration
cluster.routing.allocation.awareness.attributes: rack_id,zone

# Node configuration
node.attr.rack_id: rack-1
node.attr.zone: us-east-1a

```

**Allocation strategies:**

```python
def design_allocation_strategy(node_count, zones, rack_per_zone):
    """Design allocation strategy for high availability"""

    total_racks = zones * rack_per_zone

    strategy = {
        'zone_awareness': {
            'enabled': zones > 1,
            'forced_awareness': zones > 2,  # Force cross-zone allocation
            'min_master_nodes': (zones // 2) + 1
        },
        'rack_awareness': {
            'enabled': rack_per_zone > 1,
            'allocation_filtering': True
        },
        'shard_allocation': {
            'same_shard_threshold': 2,  # Max same shard per rack
            'total_shards_per_node': node_count // 3,  # Conservative limit
            'disk_watermark_low': '85%',
            'disk_watermark_high': '90%'
        }
    }

    return strategy

```

### Rollover and Shard Count Adjustment

**Dynamic shard count based on growth:**

```python
def calculate_rollover_shards(current_write_rate, projected_growth, time_period_days):
    """Calculate optimal shard count for rollover indices"""

    # Current daily write volume
    daily_docs = current_write_rate * 24 * 3600  # docs/sec to docs/day
    daily_size_gb = daily_docs * 0.001  # Assume 1KB per doc average

    # Project future growth
    growth_factor = 1 + (projected_growth * time_period_days / 365)
    future_daily_size_gb = daily_size_gb * growth_factor

    # Calculate shards needed for time period
    period_size_gb = future_daily_size_gb * time_period_days
    target_shard_size_gb = 30  # Target shard size

    optimal_shards = max(1, int(period_size_gb / target_shard_size_gb))

    # Ensure power of 2 for even distribution (optional)
    power_of_2_shards = 2 ** (optimal_shards - 1).bit_length()

    return {
        'calculated_shards': optimal_shards,
        'power_of_2_shards': power_of_2_shards,
        'period_size_gb': period_size_gb,
        'projected_shard_size_gb': period_size_gb / optimal_shards
    }

```

---

## 🔍 Monitoring and Troubleshooting Shards

### Shard Health Metrics

**Key metrics to monitor:**

| Metric | Normal Range | Warning Signs | Actions |
| --- | --- | --- | --- |
| **Shard size** | 10-50GB | >100GB or <1GB | Adjust shard count or rollover policy |
| **Documents per shard** | 1M-50M | >100M or <100K | Rebalance or resize |
| **Query latency per shard** | <50ms | >200ms | Investigate hot shards or slow queries |
| **Indexing rate per shard** | Varies | Significant skew | Check routing and document distribution |
| **Memory usage per shard** | 1-10MB | >20MB | Consider segment merging or field optimization |

**Shard distribution analysis:**

```python
def analyze_shard_distribution(shard_stats):
    """Analyze shard distribution for health and balance"""

    analysis = {
        'total_shards': len(shard_stats),
        'size_distribution': {},
        'document_distribution': {},
        'performance_metrics': {}
    }

    sizes = [shard['size_gb'] for shard in shard_stats]
    doc_counts = [shard['doc_count'] for shard in shard_stats]

    analysis['size_distribution'] = {
        'min_gb': min(sizes),
        'max_gb': max(sizes),
        'avg_gb': sum(sizes) / len(sizes),
        'std_dev': calculate_std_dev(sizes),
        'coefficient_of_variation': calculate_std_dev(sizes) / (sum(sizes) / len(sizes))
    }

    # Coefficient of variation > 0.3 indicates significant imbalance
    if analysis['size_distribution']['coefficient_of_variation'] > 0.3:
        analysis['recommendations'] = [
            'Consider custom routing to balance shard sizes',
            'Review document ID patterns for hotspots',
            'Monitor individual shard performance'
        ]

    return analysis

```

### Common Shard Problems and Solutions

**Problem 1: Shard Skew**

```python
# Symptoms
shard_sizes = {
    'shard_0': 80.5,  # Hot shard
    'shard_1': 45.2,
    'shard_2': 42.8,
    'shard_3': 38.9,
    'shard_4': 41.1
}

# Diagnosis
avg_size = sum(shard_sizes.values()) / len(shard_sizes)  # 49.7GB
max_deviation = max(shard_sizes.values()) - avg_size      # 30.8GB
deviation_percentage = (max_deviation / avg_size) * 100  # 62%

# Solutions
solutions = [
    'Analyze document ID patterns causing hotspots',
    'Implement custom routing with composite keys',
    'Consider reindexing with modified document IDs',
    'Add randomness to time-based document IDs'
]

```

**Problem 2: Too Many Small Shards**

```python
# Symptoms
cluster_stats = {
    'total_indices': 100,
    'total_shards': 2000,  # 20 shards per index
    'avg_shard_size_gb': 2.5,  # Very small shards
    'coordination_overhead': 'high',
    'memory_per_shard_mb': 15  # High overhead ratio
}

# Solutions
consolidation_strategy = {
    'reduce_shard_count': {
        'target_shards_per_index': 5,
        'target_shard_size_gb': 10,
        'requires_reindexing': True
    },
    'index_consolidation': {
        'merge_small_indices': True,
        'use_index_patterns': True,
        'implement_rollover': True
    }
}

```

**Problem 3: Recovery Performance Issues**

```python
def estimate_recovery_time(shard_size_gb, network_bandwidth_mbps=1000, disk_io_mbps=500):
    """Estimate shard recovery time"""

    # Convert to consistent units
    shard_size_mb = shard_size_gb * 1024

    # Bottleneck is usually disk I/O or network
    bottleneck_mbps = min(network_bandwidth_mbps, disk_io_mbps)

    # Add overhead for coordination and verification
    effective_throughput = bottleneck_mbps * 0.7  # 30% overhead

    recovery_time_minutes = shard_size_mb / effective_throughput / 60

    recommendations = []
    if recovery_time_minutes > 30:
        recommendations.append('Consider smaller shard sizes for faster recovery')
    if recovery_time_minutes > 60:
        recommendations.append('Implement shard allocation awareness for zone failures')

    return {
        'estimated_recovery_minutes': recovery_time_minutes,
        'bottleneck': 'disk_io' if disk_io_mbps < network_bandwidth_mbps else 'network',
        'recommendations': recommendations
    }

```

---

## 🎛️ Shard Strategy Decision Framework

### Choosing Your Shard Strategy

**Decision tree for shard count:**

```
Start: What's your primary use case?

├── Search-heavy workload
│   ├── Small dataset (<100GB) → 1-3 shards
│   ├── Medium dataset (100GB-1TB) → 5-15 shards
│   └── Large dataset (>1TB) → 20+ shards
│
├── Write-heavy workload
│   ├── Even write distribution → Standard sharding
│   ├── Skewed write patterns → Custom routing
│   └── Time-series data → Date-based indices + rollover
│
├── Analytics workload
│   ├── Interactive queries → Many small shards (10-30GB)
│   ├── Batch processing → Fewer large shards (50-200GB)
│   └── Mixed workload → Balanced approach (30-50GB)
│
└── Multi-tenant application
    ├── Few large tenants → Custom routing per tenant
    ├── Many small tenants → Standard sharding
    └── Mixed tenant sizes → Hybrid routing strategy

```

### Validation Checklist

**Before implementing shard strategy:**

- [ ]  **Estimated data growth over 2 years**
- [ ]  **Peak write throughput requirements**
- [ ]  **Query patterns and performance SLAs**
- [ ]  **Recovery time objectives (RTO)**
- [ ]  **Node failure scenarios planned**
- [ ]  **Monitoring and alerting configured**
- [ ]  **Rollback plan if strategy fails**

---

## 💡 Key Takeaways

✅ **Shard count is immutable—plan for future growth from day one**

✅ **Balance shard size between query performance and operational complexity**

✅ **Document ID design directly impacts shard distribution and performance**

✅ **Custom routing can solve specific performance problems but adds complexity**

✅ **Monitor shard distribution and performance continuously**

✅ **Recovery time increases with shard size—plan for failure scenarios**

✅ **Time-based indices with rollover often provide better shard management than static indices**

---