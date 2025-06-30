# Troubleshooting Guide

## Systematic Elasticsearch Problem Resolution

This comprehensive troubleshooting guide provides systematic approaches to diagnosing and resolving common Elasticsearch issues. Organized by symptoms and problem categories, each section includes diagnostic steps, root cause analysis, and proven solutions.

---

## 🎯 Troubleshooting Methodology

### The DETECT Framework

Use this systematic approach for any Elasticsearch issue:

| Step | Action | Purpose | Tools |
| --- | --- | --- | --- |
| **D**efine | Clearly describe the problem | Understand scope and impact | User reports, monitoring alerts |
| **E**xamine | Gather diagnostic information | Collect relevant data | Health APIs, logs, metrics |
| **T**race | Identify the root cause | Find the underlying issue | Analysis tools, correlation |
| **E**liminate | Implement the fix | Resolve the problem | Configuration changes, scaling |
| **C**onfirm | Verify the solution works | Ensure problem is resolved | Testing, monitoring |
| **T**une | Optimize to prevent recurrence | Improve system resilience | Monitoring, alerting, documentation |

### Diagnostic Information Collection

**Essential data to gather for any issue:**

```bash
# Cluster overview
GET /_cluster/health?level=indices&pretty
GET /_cat/nodes?v&h=name,node.role,heap.percent,cpu,load_1m,disk.used_percent
GET /_cat/indices?v&h=index,health,pri,rep,docs.count,store.size&s=store.size:desc

# Performance metrics
GET /_nodes/stats/indices,os,jvm,process?human
GET /_nodes/hot_threads?threads=5&ignore_idle_threads=true
GET /_tasks?detailed=true&group_by=parents

# Resource utilization
GET /_cat/allocation?v&h=node,disk.used_percent,disk.avail&s=disk.used_percent:desc
GET /_cat/thread_pool?v&h=name,active,queue,rejected&s=rejected:desc
GET /_cat/recovery?v&active_only=true

```

---

## 🚨 Critical Issues (Red Cluster Status)

### Symptom: Cluster Status Red

**Immediate Impact:** Data unavailable, potential data loss risk

### Diagnostic Steps

```bash
# 1. Check which indices are affected
GET /_cat/indices?v&h=index,health,status,pri,rep&s=health

# 2. Identify unassigned shards
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason,node&s=state

# 3. Check allocation explanation
GET /_cluster/allocation/explain
{
  "index": "affected_index",
  "shard": 0,
  "primary": true
}

# 4. Review recent cluster events
GET /_cat/recovery?v&h=index,shard,time,type,stage,source_node,target_node

```

### Common Causes and Solutions

**1. Node Failure**

```bash
# Symptoms
- Node disappeared from cluster
- Primary shards unassigned
- Replicas being promoted

# Diagnosis
GET /_cat/nodes?v
# Missing nodes indicate failure

# Solutions
# Option A: Restart failed node
sudo systemctl restart elasticsearch

# Option B: Remove failed node permanently
POST /_cluster/voting_config_exclusions?node_names=failed_node_name

```

**2. Disk Watermark Exceeded**

```bash
# Symptoms
- "disk usage exceeded flood-stage watermark" errors
- Shards relocated away from nodes

# Diagnosis
GET /_cat/allocation?v&h=node,disk.used_percent,disk.avail
GET /_cluster/settings?include_defaults=true&filter_path=*.cluster.routing.allocation.disk.*

# Solutions
# Immediate: Increase disk watermark threshold (temporary)
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low": "90%",
    "cluster.routing.allocation.disk.watermark.high": "95%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "98%"
  }
}

# Long-term: Add storage or delete old data
DELETE /old_index_*
POST /index_name/_forcemerge?max_num_segments=1

```

**3. Split Brain Prevention**

```bash
# Symptoms
- Multiple master nodes
- Cluster forms separate clusters

# Diagnosis
GET /_cat/master?v
# Should show only one master

# Solutions
# Ensure odd number of master-eligible nodes
# Check discovery configuration
GET /_cluster/settings?include_defaults=true&filter_path=*.discovery.*

# Fix in elasticsearch.yml
discovery.seed_hosts: ["node1", "node2", "node3"]
cluster.initial_master_nodes: ["node1", "node2", "node3"]

```

---

## ⚠️ Performance Issues

### Symptom: Slow Query Performance

**Impact:** Poor user experience, timeouts, increased resource usage

### Diagnostic Process

```bash
# 1. Identify slow queries
GET /_cat/thread_pool?v&h=name,active,queue,rejected&s=name
GET /_nodes/hot_threads?threads=5&type=wait

# 2. Check for circuit breaker trips
GET /_nodes/stats/breaker?human

# 3. Analyze query patterns
# Enable slow query logging first
PUT /index_name/_settings
{
  "index.search.slowlog.threshold.query.warn": "10s",
  "index.search.slowlog.threshold.query.info": "5s",
  "index.search.slowlog.threshold.query.debug": "2s",
  "index.search.slowlog.threshold.query.trace": "500ms"
}

```

### Common Performance Problems

**1. Inefficient Queries**

```json
// Problem: Using query context for filtering
{
  "query": {
    "bool": {
      "must": [
        {"match": {"title": "search term"}},
        {"term": {"status": "published"}},  // Should be in filter
        {"range": {"date": {"gte": "2023-01-01"}}}  // Should be in filter
      ]
    }
  }
}

// Solution: Use filter context
{
  "query": {
    "bool": {
      "must": [
        {"match": {"title": "search term"}}
      ],
      "filter": [
        {"term": {"status": "published"}},
        {"range": {"date": {"gte": "2023-01-01"}}}
      ]
    }
  }
}

```

**2. Large Result Sets**

```bash
# Problem: Deep pagination with from/size
GET /index/_search
{
  "from": 10000,
  "size": 100  // Expensive!
}

# Solution: Use search_after
GET /index/_search
{
  "size": 100,
  "sort": [{"timestamp": "desc"}, {"_id": "asc"}],
  "search_after": ["2023-10-15T14:30:00Z", "doc_123"]
}

```

**3. Expensive Aggregations**

```json
// Problem: High cardinality aggregation
{
  "aggs": {
    "users": {
      "terms": {
        "field": "user_id",
        "size": 100000  // Too many buckets!
      }
    }
  }
}

// Solution: Use composite aggregation
{
  "aggs": {
    "users": {
      "composite": {
        "size": 1000,
        "sources": [
          {"user": {"terms": {"field": "user_id"}}}
        ]
      }
    }
  }
}

```

### Symptom: High Memory Usage

**Impact:** OutOfMemoryError, garbage collection pauses, cluster instability

### Memory Diagnostic Commands

```bash
# Check heap usage
GET /_cat/nodes?v&h=name,heap.percent,heap.current,heap.max,ram.percent

# Detailed JVM stats
GET /_nodes/stats/jvm?human

# Check circuit breakers
GET /_nodes/stats/breaker?human

# Field data usage
GET /_cat/fielddata?v&h=node,field,size&s=size:desc

# Segment memory
GET /_cat/segments?v&h=index,shard,segment,size,memory&s=memory:desc

```

### Memory Issues Resolution

**1. Heap Size Optimization**

```bash
# Check current heap configuration
GET /_nodes/jvm?filter_path=**.mem

# Optimal heap size (in jvm.options)
# Never exceed 32GB (compressed OOPs limit)
-Xms16g
-Xmx16g

# For machines with >64GB RAM, consider multiple nodes

```

**2. Field Data Circuit Breaker**

```bash
# Symptoms
GET /_nodes/stats/breaker
# Look for "fielddata" breaker trips

# Solutions
# Option A: Increase breaker limit (temporary)
PUT /_cluster/settings
{
  "transient": {
    "indices.breaker.fielddata.limit": "50%"
  }
}

# Option B: Use doc_values (permanent fix)
PUT /index/_mapping
{
  "properties": {
    "category": {
      "type": "keyword",
      "doc_values": true  // Default for keyword fields
    }
  }
}

# Option C: Clear fielddata cache
POST /_cache/clear?fielddata=true

```

**3. Segment Memory Issues**

```bash
# Check segment count
GET /_cat/segments?v&h=index,shard,segment,size&s=index

# Force merge to reduce segments
POST /index_name/_forcemerge?max_num_segments=1

# Prevent future issues with better refresh settings
PUT /index_name/_settings
{
  "refresh_interval": "30s"  // Less frequent refreshes
}

```

---

## 🐌 Indexing Performance Issues

### Symptom: Slow Indexing Speed

**Impact:** Data ingestion delays, queue buildup, real-time data lag

### Indexing Diagnostics

```bash
# Check indexing performance
GET /_cat/thread_pool?v&h=name,active,queue,rejected&s=rejected:desc

# Indexing stats
GET /_stats/indexing?human

# Merge operations
GET /_cat/segments?v&h=index,shard,segment,size&s=size:desc

# Refresh operations
GET /_stats/refresh?human

```

### Indexing Optimization Strategies

**1. Bulk Indexing Optimization**

```json
// Optimal bulk request structure
POST /_bulk
{"index": {"_index": "logs", "_id": "1"}}
{"@timestamp": "2023-10-15T14:30:00Z", "message": "log entry"}
{"index": {"_index": "logs", "_id": "2"}}
{"@timestamp": "2023-10-15T14:30:00Z", "message": "another entry"}

// Optimal bulk sizing
{
  "bulk_size": "5-15MB",  // Target size per bulk request
  "concurrent_requests": "2-4",  // Parallel bulk requests
  "queue_capacity": "200"  // Index queue size
}

```

**2. Index Settings for High Throughput**

```json
PUT /high_volume_index
{
  "settings": {
    "number_of_replicas": 0,  // Disable during initial load
    "refresh_interval": "-1",  // Disable refresh during bulk load
    "number_of_shards": 6,  // Based on anticipated size
    "translog.flush_threshold_size": "1gb",
    "translog.sync_interval": "30s"
  }
}

// After bulk loading, restore normal settings
PUT /high_volume_index/_settings
{
  "number_of_replicas": 1,
  "refresh_interval": "1s"
}

// Manual refresh after bulk load
POST /high_volume_index/_refresh

```

**3. Merge Policy Tuning**

```json
PUT /index_name/_settings
{
  "index": {
    "merge.policy.max_merged_segment": "10gb",
    "merge.policy.segments_per_tier": 5,
    "merge.scheduler.max_thread_count": 2
  }
}

```

---

## 🔍 Search Issues

### Symptom: Incorrect Search Results

**Impact:** Poor user experience, missed relevant documents

### Search Accuracy Diagnostics

```bash
# Validate query syntax
GET /index/_validate/query?explain=true
{
  "query": {"your_query_here"}
}

# Explain scoring
GET /index/_search
{
  "explain": true,
  "query": {"your_query_here"}
}

# Analyze text processing
GET /_analyze
{
  "analyzer": "standard",
  "text": "Your search text"
}

```

### Common Search Problems

**1. Analyzer Mismatch**

```bash
# Problem: Search and index use different analyzers
# Check current mapping
GET /index/_mapping

# Solution: Ensure consistent analysis
PUT /index/_mapping
{
  "properties": {
    "title": {
      "type": "text",
      "analyzer": "standard",
      "search_analyzer": "standard"  // Explicit search analyzer
    }
  }
}

```

**2. Relevance Issues**

```json
// Problem: Poor relevance scoring
{
  "query": {
    "multi_match": {
      "query": "search terms",
      "fields": ["title", "description"]  // No field boosting
    }
  }
}

// Solution: Add field boosting and tune scoring
{
  "query": {
    "multi_match": {
      "query": "search terms",
      "fields": ["title^3", "description^1"],
      "type": "best_fields",
      "tie_breaker": 0.3,
      "minimum_should_match": "75%"
    }
  }
}

```

**3. Missing Results**

```bash
# Check if documents exist
GET /index/_count
{
  "query": {"term": {"_id": "expected_doc_id"}}
}

# Check field mapping
GET /index/_mapping/field/field_name

# Verify analysis chain
GET /index/_analyze
{
  "field": "field_name",
  "text": "search term"
}

```

---

## 🔧 Cluster Management Issues

### Symptom: Shard Allocation Problems

**Impact:** Unbalanced cluster, poor performance, reduced availability

### Shard Allocation Diagnostics

```bash
# Check shard distribution
GET /_cat/shards?v&h=index,shard,prirep,state,docs,store,node&s=node

# Allocation settings
GET /_cluster/settings?include_defaults=true&filter_path=*.cluster.routing.*

# Allocation explanation
GET /_cluster/allocation/explain

```

### Allocation Issues and Solutions

**1. Shard Rebalancing**

```bash
# Enable/disable rebalancing
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.rebalance.enable": "all"  // none, primaries, replicas, all
  }
}

# Manual reroute
POST /_cluster/reroute
{
  "commands": [
    {
      "move": {
        "index": "index_name",
        "shard": 0,
        "from_node": "node1",
        "to_node": "node2"
      }
    }
  ]
}

```

**2. Allocation Awareness**

```bash
# Configure allocation awareness (in elasticsearch.yml)
cluster.routing.allocation.awareness.attributes: zone
node.attr.zone: zone1

# Force awareness
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.awareness.force.zone.values": ["zone1", "zone2"]
  }
}

```

### Symptom: Master Node Issues

**Impact:** Cluster instability, split brain risk

### Master Node Diagnostics

```bash
# Check master status
GET /_cat/master?v

# Master node stats
GET /_nodes/_master/stats

# Pending cluster tasks
GET /_cat/pending_tasks?v&h=insertOrder,timeInQueue,priority,source&s=insertOrder

```

### Master Node Solutions

**1. Master Election Issues**

```bash
# Check discovery configuration
GET /_cluster/settings?include_defaults=true&filter_path=*.discovery.*

# Exclude problematic nodes from voting
POST /_cluster/voting_config_exclusions?node_names=problem_node

# Clear voting exclusions
DELETE /_cluster/voting_config_exclusions

```

**2. Dedicated Master Nodes**

```yaml
# In elasticsearch.yml for dedicated master nodes
node.roles: ["master"]
discovery.seed_hosts: ["master1", "master2", "master3"]
cluster.initial_master_nodes: ["master1", "master2", "master3"]

```

---

## 📊 Monitoring and Alerting Setup

### Essential Monitoring Queries

```bash
# Cluster health monitoring
GET /_cluster/health

# Performance monitoring
GET /_nodes/stats/indices,os,jvm,process

# Index health
GET /_cat/indices?v&h=index,health,pri,rep,docs.count,store.size&s=health

# Search performance
GET /_nodes/stats/indices/search

# Indexing performance
GET /_nodes/stats/indices/indexing

```

### Proactive Monitoring Alerts

```python
def setup_monitoring_alerts():
    """Essential monitoring alerts for Elasticsearch"""

    alerts = {
        'cluster_health': {
            'red_status': {
                'condition': 'cluster status = red',
                'severity': 'critical',
                'action': 'immediate investigation'
            },
            'yellow_status': {
                'condition': 'cluster status = yellow for > 5 minutes',
                'severity': 'warning',
                'action': 'check shard allocation'
            }
        },

        'performance': {
            'high_query_latency': {
                'condition': 'P95 query latency > 1000ms',
                'severity': 'warning',
                'action': 'analyze slow queries'
            },
            'high_indexing_latency': {
                'condition': 'indexing latency > 5000ms',
                'severity': 'warning',
                'action': 'check indexing performance'
            }
        },

        'resources': {
            'high_heap_usage': {
                'condition': 'heap usage > 85%',
                'severity': 'critical',
                'action': 'scale cluster or optimize queries'
            },
            'high_disk_usage': {
                'condition': 'disk usage > 85%',
                'severity': 'warning',
                'action': 'add storage or implement ILM'
            }
        }
    }

    return alerts

```

---

## 🚨 Emergency Procedures

### Complete Cluster Failure Recovery

```bash
# 1. Assess the situation
curl -X GET "localhost:9200/_cluster/health?pretty"

# 2. Start nodes in order (master-eligible first)
sudo systemctl start elasticsearch

# 3. Check cluster formation
curl -X GET "localhost:9200/_cat/nodes?v"

# 4. If cluster won't form, check discovery
curl -X GET "localhost:9200/_cluster/settings?pretty"

# 5. Emergency cluster bootstrap (last resort)
# Only if all master nodes are down
elasticsearch-node unsafe-bootstrap

```

### Data Recovery Procedures

```bash
# 1. Restore from snapshot
POST /_snapshot/repository_name/snapshot_name/_restore
{
  "indices": "lost_index",
  "ignore_unavailable": true,
  "include_global_state": false
}

# 2. Check restore progress
GET /_cat/recovery?v&h=index,shard,time,type,stage,source_node,target_node

# 3. Verify data integrity
GET /restored_index/_count
GET /restored_index/_search?size=0

```

---

## 📋 Troubleshooting Checklist

### Pre-Issue Prevention

✅ **Monitoring configured** for all key metrics

✅ **Alerting set up** for critical thresholds

✅ **Backup strategy** tested and verified

✅ **Capacity planning** based on growth trends

✅ **Documentation updated** with cluster configuration

✅ **Runbooks prepared** for common scenarios

### During Issue Response

✅ **Problem clearly defined** with scope and impact

✅ **Diagnostic data collected** before making changes

✅ **Changes documented** for rollback if needed

✅ **Impact assessed** before implementing solutions

✅ **Stakeholders notified** of status and ETA

✅ **Solution verified** through testing

### Post-Issue Review

✅ **Root cause identified** and documented

✅ **Prevention measures** implemented

✅ **Monitoring gaps** addressed

✅ **Documentation updated** with lessons learned

✅ **Team knowledge shared** through post-mortem

✅ **Process improvements** identified and implemented

---

## 💡 Pro Tips for Effective Troubleshooting

### Information Gathering

1. **Collect before you change** - Always gather diagnostic data before making modifications
2. **Time correlation** - Align issue timing with changes, deployments, or traffic patterns
3. **Multi-node perspective** - Check if issues are cluster-wide or node-specific
4. **Log correlation** - Cross-reference Elasticsearch logs with application and system logs

### Problem Solving

1. **Start simple** - Check obvious causes (disk space, connectivity) before complex analysis
2. **One change at a time** - Make incremental changes to identify what works
3. **Test in isolation** - Use test clusters to verify solutions when possible
4. **Document everything** - Keep detailed notes of investigation and changes

### Communication

1. **Set expectations** - Provide realistic timelines for investigation and resolution
2. **Regular updates** - Keep stakeholders informed of progress
3. **Clear escalation** - Know when to involve additional expertise or vendor support
4. **Share knowledge** - Document solutions for future reference

---
