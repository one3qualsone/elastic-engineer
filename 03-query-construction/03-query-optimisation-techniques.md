# Query Optimisation Techniques

## Maximising Performance and Relevance in Production

Query optimisation in Elasticsearch goes beyond making queries "fast"—it's about finding the right balance between response time, resource usage, relevance quality, and system stability. This guide covers proven techniques for optimising queries at scale, from micro-optimisations that save milliseconds to architectural patterns that improve system-wide performance.

---

## 🎯 Optimisation Philosophy

### The Performance-Quality-Cost Triangle

**You can optimise for any two:**

- **Fast + High Quality = Expensive** (more hardware, complex caching)
- **Fast + Cheap = Lower Quality** (simplified queries, aggressive filtering)
- **High Quality + Cheap = Slower** (complex scoring, detailed analysis)

**Real-world example:**

```json
// High quality, slower
{
  "query": {
    "function_score": {
      "query": {"multi_match": {"query": "laptop", "fields": ["name^3", "description", "features", "reviews"]}},
      "functions": [
        {"field_value_factor": {"field": "rating", "factor": 1.2}},
        {"gauss": {"price": {"origin": 800, "scale": 200}}},
        {"script_score": {"script": "Math.log(2 + doc['sales_count'].value)"}}
      ]
    }
  }
}

// Fast, simpler
{
  "query": {
    "bool": {
      "must": [{"match": {"name": "laptop"}}],
      "filter": [{"range": {"rating": {"gte": 4.0}}}]
    }
  }
}

```

### Measurement-Driven Optimisation

**What to measure:**

- Query latency (P50, P95, P99)
- Throughput (queries per second)
- Resource usage (CPU, memory, I/O)
- Cache hit rates
- Result quality metrics

**Optimisation priorities:**

1. **Correctness first** - Wrong fast results are useless
2. **Eliminate bottlenecks** - Fix the biggest problems first
3. **Measure everything** - Data-driven decisions only
4. **Test in production** - Synthetic tests miss real-world patterns

---

## ⚡ Query Structure Optimisation

### Filter vs Query Context

**The fundamental rule:** Use filters for exact matches, queries for relevance.

**Filter context (cached, no scoring):**

```json
{
  "query": {
    "bool": {
      "filter": [
        {"term": {"status": "published"}},
        {"range": {"price": {"gte": 100, "lte": 500}}},
        {"terms": {"category": ["electronics", "computers"]}}
      ]
    }
  }
}

```

**Query context (scored, not cached):**

```json
{
  "query": {
    "bool": {
      "must": [
        {"match": {"title": "laptop computer"}},
        {"match": {"description": "gaming performance"}}
      ]
    }
  }
}

```

**Hybrid approach (optimal):**

```json
{
  "query": {
    "bool": {
      "must": [
        // Scored for relevance
        {"multi_match": {"query": "gaming laptop", "fields": ["title^2", "description"]}}
      ],
      "filter": [
        // Exact requirements, cached
        {"term": {"status": "active"}},
        {"range": {"price": {"lte": 1500}}},
        {"term": {"in_stock": true}}
      ],
      "should": [
        // Relevance boosters
        {"term": {"featured": true}},
        {"range": {"rating": {"gte": 4.5}}}
      ]
    }
  }
}

```

### Query Clause Ordering

**Order matters for performance:**

```json
{
  "query": {
    "bool": {
      "filter": [
        // Most selective first (eliminates most documents)
        {"term": {"tenant_id": "specific_tenant"}},
        {"range": {"date": {"gte": "2024-01-01"}}},
        {"term": {"status": "active"}},
        // Least selective last
        {"exists": {"field": "description"}}
      ]
    }
  }
}

```

**Selectivity analysis:**

```python
def analyze_filter_selectivity(index_name):
    filters = [
        {"term": {"tenant_id": "tenant_123"}},
        {"range": {"date": {"gte": "2024-01-01"}}},
        {"term": {"status": "active"}},
        {"exists": {"field": "description"}}
    ]

    total_docs = es.count(index=index_name)['count']

    for filter_clause in filters:
        matching_docs = es.count(
            index=index_name,
            body={"query": filter_clause}
        )['count']

        selectivity = matching_docs / total_docs
        print(f"Filter {filter_clause}: {selectivity:.2%} selectivity")

    # Order filters by ascending selectivity (most selective first)

```

---

## 🚀 Search Performance Patterns

### Composite Queries for Complex Requirements

**Instead of multiple separate queries:**

```json
// ❌ Multiple round trips
GET /products/_search {"query": {"match": {"name": "laptop"}}}
GET /products/_search {"query": {"terms": {"brand": ["dell", "hp"]}}}
GET /products/_search {"aggs": {"avg_price": {"avg": {"field": "price"}}}}

```

**Use single composite query:**

```json
// ✅ Single optimised query
{
  "query": {
    "bool": {
      "must": [{"match": {"name": "laptop"}}],
      "filter": [{"terms": {"brand": ["dell", "hp"]}}]
    }
  },
  "aggs": {
    "avg_price": {"avg": {"field": "price"}},
    "brand_breakdown": {
      "terms": {"field": "brand.keyword"},
      "aggs": {
        "avg_price": {"avg": {"field": "price"}}
      }
    }
  },
  "size": 20,
  "_source": ["name", "brand", "price", "rating"]
}

```

### Pagination Optimisation

**Traditional pagination problems:**

```json
// ❌ Expensive for large offsets
{
  "from": 10000,  // Requires sorting 10,000+ documents
  "size": 20
}

```

**Search After pattern:**

```json
// ✅ Efficient for deep pagination
{
  "size": 20,
  "search_after": ["laptop", "2024-01-15", "PROD-12345"],
  "sort": [
    {"relevance_score": "desc"},
    {"created_at": "desc"},
    {"id": "asc"}  // Tie-breaker for consistency
  ]
}

```

**Cursor-based pagination implementation:**

```python
class ElasticsearchPaginator:
    def __init__(self, es, index, query, sort_fields, page_size=20):
        self.es = es
        self.index = index
        self.query = query
        self.sort_fields = sort_fields
        self.page_size = page_size

    def get_page(self, cursor=None):
        search_body = {
            "query": self.query,
            "sort": self.sort_fields,
            "size": self.page_size,
            "_source": True
        }

        if cursor:
            search_body["search_after"] = cursor

        response = self.es.search(index=self.index, body=search_body)
        hits = response['hits']['hits']

        next_cursor = None
        if hits and len(hits) == self.page_size:
            # Extract sort values for next page
            next_cursor = hits[-1]['sort']

        return {
            'documents': [hit['_source'] for hit in hits],
            'next_cursor': next_cursor,
            'has_more': next_cursor is not None
        }

```

---

## 💾 Caching Strategies

### Request Cache Optimisation

**What gets cached:**

- Filter context queries
- Aggregations with deterministic results
- Queries with no user-specific elements

**Cache-friendly query design:**

```json
{
  "query": {
    "bool": {
      "must": [
        // User-specific (not cached)
        {"match": {"content": "{{user_search_terms}}"}}
      ],
      "filter": [
        // Cacheable filters
        {"term": {"status": "published"}},
        {"range": {"publish_date": {"gte": "2024-01-01"}}},
        {"term": {"language": "en"}}
      ]
    }
  }
}

```

**Cache invalidation strategy:**

```python
def build_cacheable_query(user_query, filters):
    """Separate cacheable and non-cacheable components"""

    cacheable_filters = []
    dynamic_components = []

    for filter_item in filters:
        if is_static_filter(filter_item):
            cacheable_filters.append(filter_item)
        else:
            dynamic_components.append(filter_item)

    return {
        "query": {
            "bool": {
                "must": [{"match": {"content": user_query}}],
                "filter": cacheable_filters + dynamic_components
            }
        }
    }

def is_static_filter(filter_item):
    """Determine if filter is static and cacheable"""
    static_fields = ["status", "category", "language", "content_type"]

    for field in static_fields:
        if field in str(filter_item):
            return True
    return False

```

### Application-Level Caching

**Multi-layer caching strategy:**

```python
import redis
import hashlib
import json

class SearchCache:
    def __init__(self, redis_client, ttl=300):
        self.redis = redis_client
        self.ttl = ttl

    def get_cache_key(self, query, filters, user_context=None):
        """Generate deterministic cache key"""
        cache_data = {
            'query': query,
            'filters': sorted(filters, key=str),  # Normalize order
            'user_segment': user_context.get('segment') if user_context else None
        }

        cache_string = json.dumps(cache_data, sort_keys=True)
        return f"search:{hashlib.md5(cache_string.encode()).hexdigest()}"

    def get(self, cache_key):
        """Get cached results"""
        cached_data = self.redis.get(cache_key)
        if cached_data:
            return json.loads(cached_data)
        return None

    def set(self, cache_key, results):
        """Cache search results"""
        self.redis.setex(
            cache_key,
            self.ttl,
            json.dumps(results)
        )

    def search_with_cache(self, query, filters, user_context=None):
        """Search with caching layer"""
        cache_key = self.get_cache_key(query, filters, user_context)

        # Try cache first
        cached_results = self.get(cache_key)
        if cached_results:
            return cached_results

        # Execute search
        results = self.execute_search(query, filters, user_context)

        # Cache results if cacheable
        if self.is_cacheable(query, filters):
            self.set(cache_key, results)

        return results

```

---

## 🔧 Field and Mapping Optimisations

### Selective Field Loading

**Reduce network and parsing overhead:**

```json
{
  "query": {"match": {"title": "elasticsearch"}},
  "_source": {
    "includes": ["id", "title", "summary", "url"],
    "excludes": ["full_content", "debug_info", "internal_metadata"]
  }
}

```

**Use stored fields for specific access patterns:**

```json
// Mapping
{
  "mappings": {
    "properties": {
      "title": {"type": "text", "store": true},
      "large_content": {"type": "text", "store": false}
    }
  }
}

// Query
{
  "query": {"match": {"title": "elasticsearch"}},
  "stored_fields": ["title"],
  "_source": false
}

```

### Doc Values Optimisation

**Disable doc values for fields that don't need sorting/aggregating:**

```json
{
  "mappings": {
    "properties": {
      "full_text_content": {
        "type": "text",
        "index": true,
        "doc_values": false  // Not needed for pure text search
      },
      "search_only_field": {
        "type": "keyword",
        "index": true,
        "doc_values": false,  // Disable if no aggregations needed
        "norms": false        // Disable if no relevance scoring needed
      },
      "aggregation_field": {
        "type": "keyword",
        "index": false,       // Disable if no term queries needed
        "doc_values": true    // Keep for aggregations
      }
    }
  }
}

```

---

## 📊 Aggregation Performance

### Aggregation Optimisation Patterns

**Efficient aggregation structure:**

```json
{
  "size": 0,  // Don't return documents, just aggregations
  "aggs": {
    "categories": {
      "terms": {
        "field": "category.keyword",
        "size": 10,
        "min_doc_count": 100  // Skip rare categories
      },
      "aggs": {
        "avg_price": {
          "avg": {"field": "price"}
        },
        "top_products": {
          "top_hits": {
            "size": 3,
            "_source": ["name", "price"],
            "sort": [{"sales_rank": "asc"}]
          }
        }
      }
    }
  }
}

```

**Cardinality management:**

```json
{
  "aggs": {
    "high_cardinality_field": {
      "terms": {
        "field": "user_id.keyword",
        "size": 1000,
        "execution_hint": "map",  // Use hash map for high cardinality
        "collect_mode": "breadth_first"  // Optimise for many buckets
      }
    },
    "low_cardinality_field": {
      "terms": {
        "field": "status.keyword",
        "execution_hint": "global_ordinals",  // Use ordinals for low cardinality
        "collect_mode": "depth_first"
      }
    }
  }
}

```

### Composite Aggregations for Large Result Sets

**Traditional approach (memory intensive):**

```json
{
  "aggs": {
    "large_terms": {
      "terms": {
        "field": "category.keyword",
        "size": 10000  // Memory intensive
      }
    }
  }
}

```

**Composite aggregation (memory efficient):**

```json
{
  "aggs": {
    "category_pagination": {
      "composite": {
        "size": 1000,
        "sources": [
          {"category": {"terms": {"field": "category.keyword"}}}
        ]
      }
    }
  }
}

```

---

## 🔍 Query Profiling and Analysis

### Using the Profile API

**Enable profiling:**

```json
{
  "profile": true,
  "query": {
    "bool": {
      "must": [{"match": {"title": "elasticsearch"}}],
      "filter": [{"term": {"status": "published"}}]
    }
  }
}

```

**Analyse profile results:**

```python
def analyze_query_profile(profile_result):
    """Analyze Elasticsearch query profile for optimisation opportunities"""

    shards = profile_result['profile']['shards']

    for shard_id, shard_data in enumerate(shards):
        print(f"\n--- Shard {shard_id} Analysis ---")

        searches = shard_data['searches']
        for search in searches:
            query_breakdown = search['query'][0]

            print(f"Query type: {query_breakdown['type']}")
            print(f"Total time: {query_breakdown['time_in_nanos'] / 1000000:.2f}ms")

            # Identify expensive operations
            if query_breakdown['time_in_nanos'] > 50000000:  # 50ms
                print("⚠️  Expensive query component detected")

                # Analyze breakdown
                breakdown = query_breakdown['breakdown']
                for operation, time_ns in breakdown.items():
                    if time_ns > 10000000:  # 10ms
                        print(f"  - {operation}: {time_ns / 1000000:.2f}ms")

            # Analyze children queries
            if 'children' in query_breakdown:
                analyze_children(query_breakdown['children'])

def analyze_children(children):
    """Recursively analyze child queries"""
    for child in children:
        if child['time_in_nanos'] > 20000000:  # 20ms
            print(f"  Child query {child['type']}: {child['time_in_nanos'] / 1000000:.2f}ms")

```

### Query Performance Monitoring

**Automated performance tracking:**

```python
import time
from elasticsearch import Elasticsearch

class QueryPerformanceMonitor:
    def __init__(self, es_client):
        self.es = es_client
        self.performance_data = []

    def execute_with_monitoring(self, index, query, query_id=None):
        """Execute query with performance monitoring"""

        start_time = time.time()
        start_memory = self.get_memory_usage()

        try:
            response = self.es.search(index=index, body=query)
            success = True
            error = None
        except Exception as e:
            response = None
            success = False
            error = str(e)

        end_time = time.time()
        end_memory = self.get_memory_usage()

        performance_record = {
            'query_id': query_id,
            'execution_time': (end_time - start_time) * 1000,  # ms
            'memory_delta': end_memory - start_memory,
            'took': response.get('took', 0) if response else 0,
            'hits': response['hits']['total']['value'] if response else 0,
            'timed_out': response.get('timed_out', False) if response else False,
            'success': success,
            'error': error,
            'timestamp': time.time()
        }

        self.performance_data.append(performance_record)

        # Alert on performance issues
        if performance_record['execution_time'] > 1000:  # 1 second
            self.alert_slow_query(performance_record)

        return response

    def get_memory_usage(self):
        """Get current memory usage (simplified)"""
        import psutil
        return psutil.virtual_memory().used

    def alert_slow_query(self, record):
        """Alert on slow query performance"""
        print(f"⚠️  Slow query detected: {record['execution_time']:.2f}ms")
        if record['query_id']:
            print(f"   Query ID: {record['query_id']}")

```

---

## 🏗️ Index-Level Optimisations

### Refresh Interval Tuning

**Balance real-time needs with performance:**

```python
def optimize_refresh_interval(index_name, write_load, search_requirements):
    """Dynamically adjust refresh interval based on load"""

    if write_load > 1000:  # documents per second
        if search_requirements == 'real_time':
            refresh_interval = "1s"
        elif search_requirements == 'near_real_time':
            refresh_interval = "5s"
        else:
            refresh_interval = "30s"
    else:
        refresh_interval = "1s"  # Default for low write load

    es.indices.put_settings(
        index=index_name,
        body={"refresh_interval": refresh_interval}
    )

    return refresh_interval

```

### Merge Policy Optimisation

**Optimise for your write patterns:**

```json
{
  "settings": {
    "index": {
      "merge.policy.max_merge_at_once": 5,
      "merge.policy.segments_per_tier": 5,
      "merge.policy.max_merged_segment": "5gb",
      "merge.scheduler.max_thread_count": 1
    }
  }
}

```

---

## 🎛️ Advanced Optimisation Techniques

### Query Rewriting for Performance

**Automatic query optimisation:**

```python
class QueryOptimizer:
    def __init__(self):
        self.optimization_rules = [
            self.convert_should_to_filter,
            self.extract_common_filters,
            self.reorder_clauses_by_selectivity,
            self.add_early_termination
        ]

    def optimize_query(self, query):
        """Apply optimization rules to query"""
        optimized_query = query.copy()

        for rule in self.optimization_rules:
            optimized_query = rule(optimized_query)

        return optimized_query

    def convert_should_to_filter(self, query):
        """Convert boolean should clauses to filters when appropriate"""
        if 'bool' in query.get('query', {}):
            bool_query = query['query']['bool']

            # Convert term/terms should clauses to filters
            if 'should' in bool_query:
                new_should = []
                new_filters = bool_query.get('filter', [])

                for should_clause in bool_query['should']:
                    if self.is_convertible_to_filter(should_clause):
                        new_filters.append(should_clause)
                    else:
                        new_should.append(should_clause)

                bool_query['should'] = new_should
                bool_query['filter'] = new_filters

        return query

    def is_convertible_to_filter(self, clause):
        """Check if clause can be converted to filter"""
        convertible_types = ['term', 'terms', 'range', 'exists']
        return any(clause_type in clause for clause_type in convertible_types)

```

### Adaptive Query Strategies

**Query adaptation based on data characteristics:**

```python
class AdaptiveSearchStrategy:
    def __init__(self, es_client):
        self.es = es_client
        self.index_stats = {}

    def get_optimal_query_strategy(self, index, search_params):
        """Choose optimal query strategy based on index characteristics"""

        stats = self.get_index_stats(index)

        # For small indices, use simple queries
        if stats['document_count'] < 100000:
            return self.build_simple_query(search_params)

        # For large indices with high cardinality, use filters aggressively
        if stats['field_cardinality']['category'] > 10000:
            return self.build_filter_heavy_query(search_params)

        # For text-heavy indices, optimize for relevance
        if stats['avg_document_size'] > 10000:  # Large documents
            return self.build_relevance_optimized_query(search_params)

        return self.build_balanced_query(search_params)

    def get_index_stats(self, index):
        """Get index statistics for optimization decisions"""
        if index not in self.index_stats:
            stats = self.es.indices.stats(index=index)
            self.index_stats[index] = {
                'document_count': stats['indices'][index]['total']['docs']['count'],
                'avg_document_size': stats['indices'][index]['total']['store']['size_in_bytes'] / stats['indices'][index]['total']['docs']['count'],
                'field_cardinality': self.get_field_cardinality(index)
            }

        return self.index_stats[index]

```

---

## 📈 Performance Testing and Validation

### Load Testing Framework

**Simulate realistic query patterns:**

```python
import asyncio
import aiohttp
import random
from datetime import datetime, timedelta

class ElasticsearchLoadTester:
    def __init__(self, es_url, concurrent_users=10):
        self.es_url = es_url
        self.concurrent_users = concurrent_users
        self.results = []

    async def run_load_test(self, queries, duration_minutes=5):
        """Run load test with realistic query patterns"""

        end_time = datetime.now() + timedelta(minutes=duration_minutes)

        # Create concurrent user sessions
        tasks = []
        for user_id in range(self.concurrent_users):
            task = asyncio.create_task(
                self.simulate_user_session(user_id, queries, end_time)
            )
            tasks.append(task)

        # Wait for all sessions to complete
        await asyncio.gather(*tasks)

        return self.analyze_results()

    async def simulate_user_session(self, user_id, queries, end_time):
        """Simulate realistic user search behavior"""

        async with aiohttp.ClientSession() as session:
            while datetime.now() < end_time:
                # Choose query based on realistic distribution
                query = self.choose_weighted_query(queries)

                start_time = datetime.now()

                try:
                    async with session.post(
                        f"{self.es_url}/_search",
                        json=query,
                        timeout=aiohttp.ClientTimeout(total=10)
                    ) as response:
                        result = await response.json()
                        success = response.status == 200

                except asyncio.TimeoutError:
                    success = False
                    result = {"error": "timeout"}

                end_time_req = datetime.now()

                self.results.append({
                    'user_id': user_id,
                    'query_type': query.get('query_type', 'unknown'),
                    'start_time': start_time,
                    'duration_ms': (end_time_req - start_time).total_seconds() * 1000,
                    'success': success,
                    'took': result.get('took', 0) if success else None
                })

                # Realistic pause between queries
                await asyncio.sleep(random.uniform(1, 5))

    def choose_weighted_query(self, queries):
        """Choose query based on realistic usage patterns"""
        weights = [q.get('weight', 1) for q in queries]
        return random.choices(queries, weights=weights)[0]

    def analyze_results(self):
        """Analyze load test results"""
        successful_requests = [r for r in self.results if r['success']]

        if not successful_requests:
            return {"error": "No successful requests"}

        durations = [r['duration_ms'] for r in successful_requests]
        durations.sort()

        return {
            'total_requests': len(self.results),
            'successful_requests': len(successful_requests),
            'success_rate': len(successful_requests) / len(self.results),
            'avg_response_time': sum(durations) / len(durations),
            'p50_response_time': durations[len(durations) // 2],
            'p95_response_time': durations[int(len(durations) * 0.95)],
            'p99_response_time': durations[int(len(durations) * 0.99)],
            'max_response_time': max(durations),
            'requests_per_second': len(successful_requests) / (max(r['start_time'] for r in self.results) - min(r['start_time'] for r in self.results)).total_seconds()
        }

```

---

## 💡 Key Takeaways

✅ **Use filters for exact matches and queries for relevance—they have different performance characteristics**

✅ **Structure queries to maximise cache hit rates and minimise computation**

✅ **Profile queries in production to identify real bottlenecks**

✅ **Optimise mappings and index settings for your specific access patterns**

✅ **Use search_after instead of from/size for deep pagination**

✅ **Monitor performance continuously and adapt to changing patterns**

✅ **Balance relevance quality with performance based on business requirements**

---