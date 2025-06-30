# Write vs Read Optimization

## Tuning Elasticsearch for Your Workload Characteristics

Elasticsearch performance optimization isn't one-size-fits-all. A cluster optimized for high-volume logging will perform poorly for real-time search, and vice versa. Understanding your workload characteristics and optimizing accordingly is crucial for achieving both performance and cost efficiency. This guide covers proven strategies for write-heavy, read-heavy, and mixed workloads.

---

## 📊 Understanding Workload Characteristics

### Workload Classification Framework

**Write-heavy characteristics:**

- High document ingestion rates (>1000 docs/sec per node)
- Frequent updates to existing documents
- Real-time data streams (logs, metrics, events)
- Batch data loading scenarios
- Write latency more critical than read latency

**Read-heavy characteristics:**

- High query rates (>100 queries/sec per node)
- Complex search and aggregation operations
- User-facing search applications
- Analytics and reporting workloads
- Read latency directly impacts user experience

**Mixed workload characteristics:**

- Balanced read/write ratios
- Different performance requirements by time of day
- Multiple applications with different patterns
- Need optimization for both scenarios

### Measuring Your Workload

**Key metrics to track:**

| Metric Category | Write Workload | Read Workload | Mixed Workload |
| --- | --- | --- | --- |
| **Throughput** | Documents/sec indexed | Queries/sec executed | Both metrics matter |
| **Latency** | Indexing lag time | Query response time | Different SLAs per operation |
| **Resource Usage** | CPU for indexing, I/O for writes | Memory for caches, CPU for queries | Peak usage across both |
| **Data Patterns** | Document size, update frequency | Query complexity, result size | Time-based patterns |

**Workload analysis script:**

```python
def analyze_workload_pattern(metrics_data, time_window_hours=24):
    """Analyze workload characteristics from cluster metrics"""

    analysis = {
        'indexing_rate': {
            'avg_docs_per_sec': 0,
            'peak_docs_per_sec': 0,
            'total_docs': 0
        },
        'query_rate': {
            'avg_queries_per_sec': 0,
            'peak_queries_per_sec': 0,
            'total_queries': 0
        },
        'workload_classification': '',
        'optimization_recommendations': []
    }

    # Calculate indexing metrics
    indexing_events = [m for m in metrics_data if m['type'] == 'indexing']
    if indexing_events:
        analysis['indexing_rate']['avg_docs_per_sec'] = sum(e['docs_per_sec'] for e in indexing_events) / len(indexing_events)
        analysis['indexing_rate']['peak_docs_per_sec'] = max(e['docs_per_sec'] for e in indexing_events)
        analysis['indexing_rate']['total_docs'] = sum(e['doc_count'] for e in indexing_events)

    # Calculate query metrics
    query_events = [m for m in metrics_data if m['type'] == 'search']
    if query_events:
        analysis['query_rate']['avg_queries_per_sec'] = sum(e['queries_per_sec'] for e in query_events) / len(query_events)
        analysis['query_rate']['peak_queries_per_sec'] = max(e['queries_per_sec'] for e in query_events)
        analysis['query_rate']['total_queries'] = sum(e['query_count'] for e in query_events)

    # Classify workload
    write_ratio = analysis['indexing_rate']['avg_docs_per_sec'] / (analysis['indexing_rate']['avg_docs_per_sec'] + analysis['query_rate']['avg_queries_per_sec'] + 1)

    if write_ratio > 0.7:
        analysis['workload_classification'] = 'write_heavy'
    elif write_ratio < 0.3:
        analysis['workload_classification'] = 'read_heavy'
    else:
        analysis['workload_classification'] = 'mixed'

    return analysis

```

---

## ✍️ Write Optimization Strategies

### Index Settings for Write Performance

**Optimized write settings:**

```json
{
  "settings": {
    "number_of_shards": 6,
    "number_of_replicas": 0,
    "refresh_interval": "30s",
    "index.translog.flush_threshold_size": "1gb",
    "index.translog.sync_interval": "5s",
    "index.merge.policy.max_merge_at_once": 5,
    "index.merge.policy.segments_per_tier": 5,
    "index.codec": "best_compression"
  },
  "mappings": {
    "dynamic": "strict",
    "_source": {"enabled": true},
    "properties": {
      "timestamp": {"type": "date", "format": "strict_date_optional_time"},
      "message": {"type": "text", "index": false},
      "level": {"type": "keyword"},
      "service": {"type": "keyword"}
    }
  }
}

```

**Setting explanations:**

| Setting | Write Optimization | Impact |
| --- | --- | --- |
| **refresh_interval: "30s"** | Reduces refresh overhead | 30x fewer refresh operations |
| **number_of_replicas: 0** | Eliminates replica overhead during load | 50% fewer write operations |
| **translog settings** | Optimizes transaction log performance | Reduces sync frequency |
| **merge policy** | Reduces merge pressure during writes | Fewer background merges |
| **dynamic: "strict"** | Prevents mapping explosions | Consistent performance |

### Bulk Indexing Optimization

**Optimal bulk request sizing:**

```python
def optimize_bulk_size(doc_size_kb, network_latency_ms, target_latency_ms=1000):
    """Calculate optimal bulk request size"""

    # Start with reasonable defaults
    min_bulk_size = 100
    max_bulk_size = 10000
    target_request_size_mb = 10  # Target ~10MB requests

    # Calculate docs per request based on document size
    docs_per_mb = 1024 / doc_size_kb
    optimal_docs = int(target_request_size_mb * docs_per_mb)

    # Adjust for network latency
    if network_latency_ms > 50:
        optimal_docs = min(optimal_docs * 2, max_bulk_size)  # Larger batches for high latency
    elif network_latency_ms < 10:
        optimal_docs = max(optimal_docs // 2, min_bulk_size)  # Smaller batches for low latency

    # Ensure reasonable bounds
    optimal_docs = max(min_bulk_size, min(optimal_docs, max_bulk_size))

    return {
        'docs_per_request': optimal_docs,
        'estimated_request_size_mb': (optimal_docs * doc_size_kb) / 1024,
        'requests_per_second_target': 1000 / target_latency_ms
    }

# Example usage
result = optimize_bulk_size(doc_size_kb=2.5, network_latency_ms=25)
print(f"Optimal bulk size: {result['docs_per_request']} documents")

```

**Bulk indexing best practices:**

```python
import asyncio
import aiohttp
from collections import deque
import time

class OptimizedBulkIndexer:
    def __init__(self, es_url, index_name, max_queue_size=50000):
        self.es_url = es_url
        self.index_name = index_name
        self.queue = deque()
        self.max_queue_size = max_queue_size
        self.batch_size = 1000
        self.flush_interval = 5  # seconds
        self.last_flush = time.time()

    async def add_document(self, doc):
        """Add document to indexing queue"""
        self.queue.append(doc)

        # Auto-flush conditions
        if (len(self.queue) >= self.batch_size or
            time.time() - self.last_flush > self.flush_interval or
            len(self.queue) >= self.max_queue_size):
            await self.flush()

    async def flush(self):
        """Flush queued documents to Elasticsearch"""
        if not self.queue:
            return

        # Prepare bulk request
        batch = []
        while self.queue and len(batch) < self.batch_size:
            doc = self.queue.popleft()
            batch.append({"index": {"_index": self.index_name}})
            batch.append(doc)

        if not batch:
            return

        # Send bulk request
        bulk_body = '\n'.join(json.dumps(item) for item in batch) + '\n'

        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.es_url}/_bulk",
                data=bulk_body,
                headers={'Content-Type': 'application/x-ndjson'},
                timeout=aiohttp.ClientTimeout(total=30)
            ) as response:
                if response.status != 200:
                    # Handle errors - could implement retry logic
                    error_text = await response.text()
                    print(f"Bulk indexing error: {error_text}")
                else:
                    result = await response.json()
                    if result.get('errors'):
                        self.handle_bulk_errors(result['items'])

        self.last_flush = time.time()

    def handle_bulk_errors(self, items):
        """Handle individual document errors in bulk response"""
        for item in items:
            if 'index' in item and 'error' in item['index']:
                error = item['index']['error']
                print(f"Document indexing error: {error['type']} - {error['reason']}")

```

### Write Performance Tuning

**JVM and hardware optimization:**

```bash
# JVM settings for write-heavy workloads
-Xms32g
-Xmx32g
-XX:+UseG1GC
-XX:G1HeapRegionSize=32m
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:+UnlockExperimentalVMOptions
-XX:+UseJVMCICompiler

# System-level optimizations
echo 'vm.swappiness=1' >> /etc/sysctl.conf
echo 'vm.max_map_count=262144' >> /etc/sysctl.conf

# I/O scheduler optimization for SSDs
echo noop > /sys/block/nvme0n1/queue/scheduler

```

**Node specialization for writes:**

```yaml
# Dedicated ingest nodes
node.roles: [ingest]
node.attr.workload: ingest

# Ingest node settings
ingest.geoip.downloader.enabled: false
thread_pool.write.size: 8
thread_pool.write.queue_size: 1000

# Data node settings for write workloads
node.roles: [data_hot]
indices.memory.index_buffer_size: 20%
indices.memory.min_index_buffer_size: 96mb

```

---

## 🔍 Read Optimization Strategies

### Query Performance Tuning

**Cache optimization:**

```json
{
  "settings": {
    "index.queries.cache.enabled": true,
    "index.requests.cache.enable": true,
    "index.refresh_interval": "1s",
    "index.number_of_replicas": 2,
    "index.routing.allocation.total_shards_per_node": 3
  }
}

```

**Field optimization for search:**

```json
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "english",
        "fields": {
          "keyword": {"type": "keyword"},
          "search": {
            "type": "text",
            "analyzer": "search_analyzer"
          }
        }
      },
      "category": {
        "type": "keyword",
        "eager_global_ordinals": true
      },
      "price": {
        "type": "scaled_float",
        "scaling_factor": 100
      },
      "description": {
        "type": "text",
        "index_options": "freqs",
        "norms": false
      }
    }
  }
}

```

### Memory and Caching Strategies

**Memory allocation for read workloads:**

```python
def calculate_memory_allocation(total_memory_gb, workload_type):
    """Calculate optimal memory allocation for read workloads"""

    allocations = {}

    if workload_type == 'read_heavy':
        # More memory for OS cache and field data
        allocations = {
            'jvm_heap_gb': min(32, total_memory_gb * 0.4),  # Cap at 32GB
            'os_cache_gb': total_memory_gb * 0.4,           # Large OS cache
            'system_reserve_gb': total_memory_gb * 0.1,
            'field_data_circuit_breaker': '60%',
            'request_circuit_breaker': '60%'
        }
    elif workload_type == 'write_heavy':
        # More memory for indexing buffers
        allocations = {
            'jvm_heap_gb': min(32, total_memory_gb * 0.5),
            'os_cache_gb': total_memory_gb * 0.3,
            'system_reserve_gb': total_memory_gb * 0.1,
            'index_buffer_size': '20%'
        }
    else:  # mixed
        allocations = {
            'jvm_heap_gb': min(32, total_memory_gb * 0.45),
            'os_cache_gb': total_memory_gb * 0.35,
            'system_reserve_gb': total_memory_gb * 0.1
        }

    return allocations

```

**Query result caching:**

```python
class QueryCache:
    def __init__(self, max_size=1000, ttl_seconds=300):
        self.cache = {}
        self.max_size = max_size
        self.ttl_seconds = ttl_seconds
        self.access_times = {}

    def get_cache_key(self, query, filters, user_context=None):
        """Generate cache key for query"""
        cache_parts = [
            hash(json.dumps(query, sort_keys=True)),
            hash(json.dumps(filters, sort_keys=True))
        ]

        if user_context:
            # Include user segment but not specific user ID
            cache_parts.append(hash(user_context.get('segment', 'default')))

        return str(hash(tuple(cache_parts)))

    def get(self, cache_key):
        """Get cached results if valid"""
        if cache_key in self.cache:
            cached_data, timestamp = self.cache[cache_key]

            if time.time() - timestamp < self.ttl_seconds:
                self.access_times[cache_key] = time.time()
                return cached_data
            else:
                del self.cache[cache_key]
                del self.access_times[cache_key]

        return None

    def set(self, cache_key, results):
        """Cache query results"""
        # Evict old entries if at capacity
        if len(self.cache) >= self.max_size:
            # Remove least recently used
            lru_key = min(self.access_times.keys(),
                         key=lambda k: self.access_times[k])
            del self.cache[lru_key]
            del self.access_times[lru_key]

        self.cache[cache_key] = (results, time.time())
        self.access_times[cache_key] = time.time()

```

### Index Structure for Read Performance

**Hot-warm-cold architecture:**

```python
def design_read_optimized_architecture(data_age_days, query_patterns):
    """Design index architecture for read optimization"""

    architecture = {
        'hot_tier': {
            'age_range': '0-7 days',
            'node_type': 'data_hot',
            'hardware': 'NVMe SSD, high memory',
            'replica_count': 2,
            'refresh_interval': '1s',
            'target_shard_size_gb': 20
        },
        'warm_tier': {
            'age_range': '7-90 days',
            'node_type': 'data_warm',
            'hardware': 'SATA SSD, medium memory',
            'replica_count': 1,
            'refresh_interval': '30s',
            'target_shard_size_gb': 50,
            'force_merge_segments': 1
        },
        'cold_tier': {
            'age_range': '90+ days',
            'node_type': 'data_cold',
            'hardware': 'HDD, low memory',
            'replica_count': 0,
            'refresh_interval': '5m',
            'target_shard_size_gb': 100,
            'searchable_snapshots': True
        }
    }

    # Adjust based on query patterns
    if query_patterns.get('frequent_historical_queries'):
        architecture['cold_tier']['replica_count'] = 1
        architecture['cold_tier']['hardware'] = 'SATA SSD, low memory'

    if query_patterns.get('real_time_requirements'):
        architecture['hot_tier']['refresh_interval'] = '1s'
        architecture['warm_tier']['refresh_interval'] = '5s'

    return architecture

```

---

## ⚖️ Mixed Workload Optimization

### Adaptive Configuration

**Time-based optimization:**

```python
class AdaptiveElasticsearchConfig:
    def __init__(self, es_client):
        self.es = es_client
        self.current_mode = 'balanced'

    def adjust_for_workload(self, workload_metrics):
        """Dynamically adjust settings based on current workload"""

        write_rate = workload_metrics['writes_per_second']
        read_rate = workload_metrics['queries_per_second']

        if write_rate > read_rate * 3:
            new_mode = 'write_optimized'
        elif read_rate > write_rate * 3:
            new_mode = 'read_optimized'
        else:
            new_mode = 'balanced'

        if new_mode != self.current_mode:
            self.apply_mode_settings(new_mode)
            self.current_mode = new_mode

    def apply_mode_settings(self, mode):
        """Apply settings for specific optimization mode"""

        if mode == 'write_optimized':
            settings = {
                'refresh_interval': '30s',
                'number_of_replicas': 0,
                'translog.sync_interval': '5s',
                'merge.policy.max_merge_at_once': 5
            }
        elif mode == 'read_optimized':
            settings = {
                'refresh_interval': '1s',
                'number_of_replicas': 2,
                'requests.cache.enable': True,
                'queries.cache.enabled': True
            }
        else:  # balanced
            settings = {
                'refresh_interval': '5s',
                'number_of_replicas': 1,
                'requests.cache.enable': True
            }

        # Apply settings to all indices with pattern
        for index_pattern in ['logs-*', 'metrics-*', 'search-*']:
            try:
                self.es.indices.put_settings(
                    index=index_pattern,
                    body={'settings': settings}
                )
            except Exception as e:
                print(f"Failed to update {index_pattern}: {e}")

```

### Resource Allocation Strategies

**Node role specialization:**

```yaml
# Coordinating nodes for complex queries
node.roles: []
node.attr.workload: coordinating
search.remote.connect: false

# Hot data nodes (read + recent writes)
node.roles: [data_hot, ingest]
node.attr.workload: hot_data

# Warm data nodes (read-optimized)
node.roles: [data_warm]
node.attr.workload: warm_data

# Cold data nodes (archive)
node.roles: [data_cold]
node.attr.workload: cold_data

```

**Load balancing strategies:**

```python
def design_load_balancing(query_types, write_patterns):
    """Design load balancing for mixed workloads"""

    strategy = {
        'coordinating_nodes': {
            'count': 3,
            'purpose': 'Handle complex aggregations and searches',
            'hardware': 'High CPU, medium memory',
            'routing': 'Round-robin for search requests'
        },
        'data_hot_nodes': {
            'count': 6,
            'purpose': 'Recent data, high write volume',
            'hardware': 'NVMe SSD, high memory',
            'routing': 'Hash-based for writes, included in reads'
        },
        'data_warm_nodes': {
            'count': 4,
            'purpose': 'Historical data, read-heavy',
            'hardware': 'SATA SSD, medium memory',
            'routing': 'Read queries only'
        }
    }

    # Adjust based on patterns
    if write_patterns.get('burst_writes'):
        strategy['ingest_nodes'] = {
            'count': 2,
            'purpose': 'Buffer and process write bursts',
            'hardware': 'High CPU, high memory'
        }

    return strategy

```

---

## 📊 Performance Monitoring and Optimization

### Key Performance Indicators

**Write performance metrics:**

```python
def monitor_write_performance(es_client):
    """Monitor key write performance indicators"""

    stats = es_client.cluster.stats()
    indices_stats = es_client.indices.stats()

    metrics = {
        'indexing_rate': {
            'docs_per_second': stats['indices']['indexing']['index_total'] / stats['nodes']['process']['cpu']['total_in_millis'] * 1000,
            'bytes_per_second': stats['indices']['store']['size_in_bytes'] / stats['_nodes']['total'],
            'active_indexing_threads': stats['indices']['indexing']['index_current']
        },
        'resource_utilization': {
            'index_buffer_usage': stats['indices']['segments']['index_writer_memory_in_bytes'],
            'merge_current': stats['indices']['merges']['current'],
            'translog_operations': stats['indices']['translog']['operations']
        },
        'bottlenecks': []
    }

    # Identify bottlenecks
    if metrics['resource_utilization']['merge_current'] > 5:
        metrics['bottlenecks'].append('High merge activity - consider merge policy tuning')

    if stats['indices']['indexing']['index_time_in_millis'] > stats['indices']['indexing']['index_total'] * 100:
        metrics['bottlenecks'].append('High indexing latency - check I/O and CPU')

    return metrics

```

**Read performance metrics:**

```python
def monitor_read_performance(es_client):
    """Monitor key read performance indicators"""

    stats = es_client.cluster.stats()

    metrics = {
        'query_performance': {
            'avg_query_time_ms': stats['indices']['search']['query_time_in_millis'] / max(stats['indices']['search']['query_total'], 1),
            'queries_per_second': stats['indices']['search']['query_total'] / (stats['process']['timestamp'] / 1000),
            'active_queries': stats['indices']['search']['query_current']
        },
        'cache_performance': {
            'request_cache_hit_ratio': stats['indices']['request_cache']['hit_count'] / max(stats['indices']['request_cache']['hit_count'] + stats['indices']['request_cache']['miss_count'], 1),
            'query_cache_hit_ratio': stats['indices']['query_cache']['hit_count'] / max(stats['indices']['query_cache']['hit_count'] + stats['indices']['query_cache']['miss_count'], 1),
            'fielddata_memory_mb': stats['indices']['fielddata']['memory_size_in_bytes'] / (1024 * 1024)
        },
        'recommendations': []
    }

    # Performance recommendations
    if metrics['cache_performance']['request_cache_hit_ratio'] < 0.8:
        metrics['recommendations'].append('Consider query optimization or longer cache TTL')

    if metrics['query_performance']['avg_query_time_ms'] > 100:
        metrics['recommendations'].append('Investigate slow queries and consider index optimization')

    return metrics

```

### Continuous Optimization

**Automated tuning system:**

```python
class AutoTuner:
    def __init__(self, es_client, metrics_threshold=0.1):
        self.es = es_client
        self.threshold = metrics_threshold
        self.baseline_metrics = {}
        self.optimization_history = []

    def establish_baseline(self, duration_minutes=60):
        """Establish performance baseline over time period"""
        # Collect metrics over baseline period
        # This would integrate with your monitoring system
        self.baseline_metrics = {
            'avg_query_latency_ms': 85,
            'indexing_rate_docs_per_sec': 1500,
            'cache_hit_rate': 0.82,
            'cpu_utilization': 0.65
        }

    def suggest_optimizations(self, current_metrics):
        """Suggest optimizations based on performance delta"""

        suggestions = []

        # Query performance degradation
        latency_delta = (current_metrics['avg_query_latency_ms'] -
                        self.baseline_metrics['avg_query_latency_ms']) / self.baseline_metrics['avg_query_latency_ms']

        if latency_delta > self.threshold:
            suggestions.append({
                'issue': 'Query latency increased by {:.1%}'.format(latency_delta),
                'actions': [
                    'Check for slow queries in logs',
                    'Review recent mapping changes',
                    'Consider increasing replica count',
                    'Optimize frequently used queries'
                ]
            })

        # Indexing performance degradation
        indexing_delta = (self.baseline_metrics['indexing_rate_docs_per_sec'] -
                         current_metrics['indexing_rate_docs_per_sec']) / self.baseline_metrics['indexing_rate_docs_per_sec']

        if indexing_delta > self.threshold:
            suggestions.append({
                'issue': 'Indexing rate decreased by {:.1%}'.format(indexing_delta),
                'actions': [
                    'Check merge activity and I/O utilization',
                    'Consider increasing refresh interval',
                    'Review bulk request sizes',
                    'Monitor for mapping conflicts'
                ]
            })

        return suggestions

    def apply_safe_optimizations(self, suggestions):
        """Apply optimizations that are safe to automate"""

        for suggestion in suggestions:
            if 'slow queries' in suggestion['issue']:
                # Enable slow query logging
                self.es.cluster.put_settings(body={
                    'persistent': {
                        'logger.org.elasticsearch.index.search.slowlog.query': 'DEBUG',
                        'logger.org.elasticsearch.index.search.slowlog.fetch': 'DEBUG'
                    }
                })

            # Record optimization attempt
            self.optimization_history.append({
                'timestamp': time.time(),
                'issue': suggestion['issue'],
                'action_taken': 'automated_logging_enabled'
            })

```

---

## 🎯 Real-World Optimization Examples

### Example 1: E-commerce Search Platform

**Scenario:** High-traffic e-commerce site with product search

**Workload characteristics:**

- 95% read queries (product searches)
- 5% writes (product updates, inventory changes)
- Peak: 10,000 queries/second
- Complex aggregations for faceted search

**Optimization strategy:**

```python
ecommerce_config = {
    'architecture': {
        'coordinating_nodes': 3,  # Handle complex aggregations
        'data_hot_nodes': 4,      # Recent products, high memory
        'data_warm_nodes': 6,     # Historical data, read replicas
    },
    'index_settings': {
        'number_of_replicas': 3,  # High availability
        'refresh_interval': '1s', # Real-time product updates
        'requests_cache_enable': True,
        'queries_cache_enabled': True
    },
    'mapping_optimizations': {
        'category_field': 'eager_global_ordinals',  # Fast faceting
        'price_field': 'scaled_float',              # Precise pricing
        'description_field': 'index_options: freqs' # Faster text search
    }
}

```

### Example 2: Log Analytics Platform

**Scenario:** Centralized logging for microservices

**Workload characteristics:**

- 90% writes (log ingestion)
- 10% reads (troubleshooting, dashboards)
- Peak: 50,000 events/second
- Time-series data with retention

**Optimization strategy:**

```python
logging_config = {
    'architecture': {
        'ingest_nodes': 4,        # Handle burst writes
        'data_hot_nodes': 8,      # Recent logs, fast I/O
        'data_warm_nodes': 4,     # Historical logs
        'data_cold_nodes': 2,     # Archive storage
    },
    'index_settings': {
        'number_of_replicas': 0,  # Reduce write overhead
        'refresh_interval': '30s', # Batch refreshes
        'codec': 'best_compression', # Save storage
        'rollover_max_age': '1d'  # Daily indices
    },
    'lifecycle_policy': {
        'hot_phase': '7d',
        'warm_phase': '30d',
        'cold_phase': '90d',
        'delete_phase': '365d'
    }
}

```

### Example 3: Real-time Analytics Dashboard

**Scenario:** Business intelligence dashboard with real-time metrics

**Workload characteristics:**

- 60% reads (dashboard queries, reports)
- 40% writes (metric updates)
- Complex aggregations across time periods
- Sub-second query requirements

**Optimization strategy:**

```python
analytics_config = {
    'architecture': {
        'coordinating_nodes': 2,  # Complex aggregations
        'data_hot_nodes': 6,      # Balanced read/write
        'master_nodes': 3,        # Cluster stability
    },
    'index_settings': {
        'number_of_replicas': 1,  # Balance availability/performance
        'refresh_interval': '5s', # Near real-time
        'pre_computed_aggregations': True,
        'rollover_max_size': '30gb'
    },
    'query_optimizations': {
        'result_caching': True,
        'dashboard_refresh_staggering': True,
        'aggregation_pre_computation': True
    }
}

```

---

## 💡 Key Takeaways

✅ **Identify your workload characteristics before optimizing—write-heavy and read-heavy clusters need different approaches**

✅ **Use appropriate index settings for your workload—refresh intervals, replica counts, and merge policies matter**

✅ **Specialize nodes for different workload types—coordinating, ingest, hot data, warm data, and cold data nodes**

✅ **Monitor performance continuously and adjust based on changing patterns**

✅ **Cache aggressively for read workloads, batch efficiently for write workloads**

✅ **Use time-based indices and ILM for mixed workloads with data aging patterns**

✅ **Test optimizations in non-production environments before applying to production**

---
