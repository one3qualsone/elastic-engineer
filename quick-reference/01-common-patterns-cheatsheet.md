# Common Patterns Cheatsheet

## Essential Elasticsearch Queries and Configurations

This comprehensive cheatsheet provides copy-paste ready examples for the most common Elasticsearch patterns. Organized by use case, each pattern includes practical examples and common variations.

---

## 🔍 Essential Query Patterns

### Basic Search Queries

```json
// Simple text search
GET /index/_search
{
  "query": {
    "match": {
      "title": "search terms"
    }
  }
}

// Multi-field search with boosting
GET /index/_search
{
  "query": {
    "multi_match": {
      "query": "search terms",
      "fields": ["title^3", "description^1", "tags^2"],
      "type": "best_fields",
      "fuzziness": "AUTO"
    }
  }
}

// Exact value matching
GET /index/_search
{
  "query": {
    "term": {
      "status.keyword": "published"
    }
  }
}

// Multiple exact values (OR)
GET /index/_search
{
  "query": {
    "terms": {
      "category.keyword": ["electronics", "books", "clothing"]
    }
  }
}

```

### Boolean Logic Patterns

```json
// Must, Should, Must Not pattern
GET /index/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"title": "elasticsearch"}}
      ],
      "should": [
        {"match": {"description": "tutorial"}},
        {"term": {"tags": "beginner"}}
      ],
      "must_not": [
        {"term": {"status": "draft"}}
      ],
      "filter": [
        {"range": {"created_date": {"gte": "2023-01-01"}}},
        {"term": {"published": true}}
      ],
      "minimum_should_match": 1
    }
  }
}

// Nested boolean queries
GET /index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "bool": {
            "should": [
              {"match": {"title": "data science"}},
              {"match": {"title": "machine learning"}}
            ]
          }
        }
      ],
      "filter": [
        {"range": {"difficulty_level": {"gte": 3, "lte": 7}}}
      ]
    }
  }
}

```

### Range and Date Queries

```json
// Date range queries
GET /logs/_search
{
  "query": {
    "range": {
      "@timestamp": {
        "gte": "now-1d",
        "lte": "now"
      }
    }
  }
}

// Numeric range
GET /products/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 100,
        "lte": 500
      }
    }
  }
}

// Date math examples
GET /index/_search
{
  "query": {
    "range": {
      "date": {
        "gte": "now-1M/M",        // Start of last month
        "lte": "now-1M/M+1M-1d"   // End of last month
      }
    }
  }
}

```

### Wildcard and Pattern Matching

```json
// Wildcard search (use sparingly)
GET /index/_search
{
  "query": {
    "wildcard": {
      "filename.keyword": "*.pdf"
    }
  }
}

// Prefix search (more efficient)
GET /index/_search
{
  "query": {
    "prefix": {
      "username.keyword": "admin"
    }
  }
}

// Fuzzy search for typos
GET /index/_search
{
  "query": {
    "fuzzy": {
      "title": {
        "value": "elasticsearh",
        "fuzziness": "AUTO"
      }
    }
  }
}

```

---

## 📊 Aggregation Patterns

### Metrics Aggregations

```json
// Basic statistics
GET /sales/_search
{
  "size": 0,
  "aggs": {
    "sales_stats": {
      "stats": {
        "field": "amount"
      }
    },
    "total_revenue": {
      "sum": {"field": "amount"}
    },
    "avg_order_value": {
      "avg": {"field": "amount"}
    }
  }
}

// Percentiles analysis
GET /response-times/_search
{
  "size": 0,
  "aggs": {
    "response_time_percentiles": {
      "percentiles": {
        "field": "response_time_ms",
        "percents": [50, 90, 95, 99, 99.9]
      }
    },
    "sla_compliance": {
      "percentile_ranks": {
        "field": "response_time_ms",
        "values": [200, 500, 1000]
      }
    }
  }
}

// Unique value counting
GET /users/_search
{
  "size": 0,
  "aggs": {
    "unique_users": {
      "cardinality": {
        "field": "user_id.keyword"
      }
    }
  }
}

```

### Bucket Aggregations

```json
// Terms aggregation (grouping)
GET /products/_search
{
  "size": 0,
  "aggs": {
    "categories": {
      "terms": {
        "field": "category.keyword",
        "size": 10,
        "order": {"_count": "desc"}
      },
      "aggs": {
        "avg_price": {
          "avg": {"field": "price"}
        },
        "price_stats": {
          "stats": {"field": "price"}
        }
      }
    }
  }
}

// Date histogram (time-based grouping)
GET /logs/_search
{
  "size": 0,
  "aggs": {
    "logs_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "1h",
        "time_zone": "UTC"
      },
      "aggs": {
        "error_count": {
          "filter": {
            "term": {"level": "ERROR"}
          }
        },
        "avg_response_time": {
          "avg": {"field": "response_time"}
        }
      }
    }
  }
}

// Range aggregation
GET /users/_search
{
  "size": 0,
  "aggs": {
    "age_groups": {
      "range": {
        "field": "age",
        "ranges": [
          {"key": "young", "to": 25},
          {"key": "adult", "from": 25, "to": 65},
          {"key": "senior", "from": 65}
        ]
      },
      "aggs": {
        "avg_income": {
          "avg": {"field": "income"}
        }
      }
    }
  }
}

```

### Advanced Aggregation Patterns

```json
// Composite aggregation for pagination
GET /sales/_search
{
  "size": 0,
  "aggs": {
    "sales_breakdown": {
      "composite": {
        "size": 100,
        "sources": [
          {"category": {"terms": {"field": "category.keyword"}}},
          {"month": {"date_histogram": {"field": "date", "calendar_interval": "month"}}}
        ]
      },
      "aggs": {
        "total_sales": {"sum": {"field": "amount"}},
        "order_count": {"value_count": {"field": "order_id"}}
      }
    }
  }
}

// Pipeline aggregations
GET /sales/_search
{
  "size": 0,
  "aggs": {
    "monthly_sales": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "month"
      },
      "aggs": {
        "total_sales": {"sum": {"field": "amount"}},
        "sales_growth": {
          "derivative": {
            "buckets_path": "total_sales"
          }
        },
        "sales_moving_avg": {
          "moving_avg": {
            "buckets_path": "total_sales",
            "window": 3,
            "model": "simple"
          }
        }
      }
    },
    "best_month": {
      "max_bucket": {
        "buckets_path": "monthly_sales>total_sales"
      }
    }
  }
}

```

---

## 🏗️ Index Management Patterns

### Index Templates

```json
// Comprehensive index template
PUT /_index_template/application-logs
{
  "index_patterns": ["app-logs-*"],
  "priority": 100,
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "refresh_interval": "10s",
      "index.lifecycle.name": "app-logs-policy",
      "index.lifecycle.rollover_alias": "app-logs",
      "analysis": {
        "analyzer": {
          "log_analyzer": {
            "type": "custom",
            "tokenizer": "standard",
            "filter": ["lowercase", "stop"]
          }
        }
      }
    },
    "mappings": {
      "properties": {
        "@timestamp": {"type": "date"},
        "level": {"type": "keyword"},
        "service": {"type": "keyword"},
        "message": {
          "type": "text",
          "analyzer": "log_analyzer",
          "fields": {
            "keyword": {"type": "keyword", "ignore_above": 256}
          }
        },
        "duration_ms": {"type": "long"},
        "user_id": {"type": "keyword"},
        "ip_address": {"type": "ip"},
        "geo_location": {"type": "geo_point"}
      }
    }
  }
}

// Component template pattern
PUT /_component_template/timestamp_component
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": {"type": "date"},
        "created_at": {"type": "date"},
        "updated_at": {"type": "date"}
      }
    }
  }
}

PUT /_component_template/common_fields
{
  "template": {
    "mappings": {
      "properties": {
        "id": {"type": "keyword"},
        "version": {"type": "integer"},
        "tags": {"type": "keyword"}
      }
    }
  }
}

```

### Index Lifecycle Management

```json
// Complete ILM policy
PUT /_ilm/policy/standard-logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "30gb",
            "max_age": "1d",
            "max_docs": 10000000
          },
          "set_priority": {"priority": 100}
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "set_priority": {"priority": 50},
          "allocate": {
            "number_of_replicas": 0,
            "include": {"data_tier": "data_warm"}
          },
          "readonly": {},
          "forcemerge": {"max_num_segments": 1}
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "set_priority": {"priority": 0},
          "searchable_snapshot": {
            "snapshot_repository": "cold_repository"
          }
        }
      },
      "delete": {
        "min_age": "365d",
        "actions": {
          "wait_for_snapshot": {
            "policy": "daily_snapshots"
          },
          "delete": {}
        }
      }
    }
  }
}

// Alias management for rollover
PUT /app-logs-000001
{
  "aliases": {
    "app-logs": {
      "is_write_index": true
    }
  }
}

// Manual rollover
POST /app-logs/_rollover
{
  "conditions": {
    "max_age": "7d",
    "max_size": "50gb",
    "max_docs": 10000000
  }
}

```

---

## 🔒 Security Patterns

### Role-Based Access Control

```json
// Create roles
PUT /_security/role/logs_reader
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["read", "view_index_metadata"],
      "field_security": {
        "grant": ["@timestamp", "level", "message", "service"]
      },
      "query": {
        "bool": {
          "must_not": {
            "term": {"level": "DEBUG"}
          }
        }
      }
    }
  ]
}

PUT /_security/role/admin_role
{
  "cluster": ["all"],
  "indices": [
    {
      "names": ["*"],
      "privileges": ["all"]
    }
  ]
}

// Create users
PUT /_security/user/log_analyst
{
  "password": "secure_password",
  "roles": ["logs_reader"],
  "full_name": "Log Analyst",
  "email": "analyst@company.com"
}

// Document-level security
PUT /_security/role/tenant_user
{
  "indices": [
    {
      "names": ["tenant-data-*"],
      "privileges": ["read", "write"],
      "query": {
        "term": {
          "tenant_id": "{{username}}"
        }
      }
    }
  ]
}

```

### API Key Management

```json
// Create API key
POST /_security/api_key
{
  "name": "monitoring_key",
  "role_descriptors": {
    "monitoring_role": {
      "cluster": ["monitor"],
      "indices": [
        {
          "names": ["metrics-*"],
          "privileges": ["read"]
        }
      ]
    }
  },
  "expiration": "30d"
}

// Create API key with restrictions
POST /_security/api_key
{
  "name": "limited_search_key",
  "role_descriptors": {
    "limited_search": {
      "indices": [
        {
          "names": ["public-data-*"],
          "privileges": ["read"],
          "query": {
            "range": {
              "@timestamp": {
                "gte": "now-7d"
              }
            }
          }
        }
      ]
    }
  }
}

```

---

## 📸 Snapshot and Backup Patterns

### Snapshot Repository Configuration

```json
// S3 repository
PUT /_snapshot/s3_backup_repo
{
  "type": "s3",
  "settings": {
    "bucket": "elasticsearch-backups",
    "region": "us-east-1",
    "base_path": "cluster-production",
    "compress": true,
    "chunk_size": "1gb",
    "server_side_encryption": true,
    "storage_class": "STANDARD_IA",
    "max_restore_bytes_per_sec": "500mb",
    "max_snapshot_bytes_per_sec": "500mb"
  }
}

// File system repository
PUT /_snapshot/fs_backup_repo
{
  "type": "fs",
  "settings": {
    "location": "/mount/backups/elasticsearch",
    "compress": true,
    "chunk_size": "1gb"
  }
}

```

### Snapshot Lifecycle Management

```json
// SLM policy
PUT /_slm/policy/daily_snapshots
{
  "schedule": "0 30 2 * * ?",
  "name": "<daily-snapshot-{now/d}>",
  "repository": "s3_backup_repo",
  "config": {
    "indices": ["logs-*", "metrics-*"],
    "ignore_unavailable": false,
    "include_global_state": true,
    "partial": false
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}

// Manual snapshot
PUT /_snapshot/s3_backup_repo/manual-backup-2023-10-15
{
  "indices": "important-data-*",
  "ignore_unavailable": true,
  "include_global_state": false,
  "metadata": {
    "taken_by": "admin",
    "taken_because": "pre-upgrade backup"
  }
}

// Restore from snapshot
POST /_snapshot/s3_backup_repo/daily-snapshot-2023-10-15/_restore
{
  "indices": "logs-2023-10-15",
  "ignore_unavailable": true,
  "include_global_state": false,
  "rename_pattern": "logs-(.+)",
  "rename_replacement": "restored-logs-$1",
  "index_settings": {
    "index.number_of_replicas": 0
  }
}

```

---

## 🔍 Advanced Search Patterns

### Function Score Queries

```json
// Business logic scoring
GET /products/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query": "smartphone",
          "fields": ["name^2", "description"]
        }
      },
      "functions": [
        {
          "filter": {"term": {"featured": true}},
          "weight": 2.0
        },
        {
          "field_value_factor": {
            "field": "popularity_score",
            "factor": 0.5,
            "modifier": "log1p",
            "missing": 1
          }
        },
        {
          "exp": {
            "release_date": {
              "origin": "now",
              "scale": "30d",
              "decay": 0.5
            }
          }
        },
        {
          "random_score": {
            "seed": 12345,
            "field": "_seq_no"
          }
        }
      ],
      "boost_mode": "multiply",
      "score_mode": "sum",
      "max_boost": 5.0,
      "min_score": 0.5
    }
  }
}

```

### Nested Object Queries

```json
// Query nested objects
GET /products/_search
{
  "query": {
    "nested": {
      "path": "reviews",
      "query": {
        "bool": {
          "must": [
            {"range": {"reviews.rating": {"gte": 4}}},
            {"match": {"reviews.comment": "excellent"}}
          ]
        }
      },
      "inner_hits": {
        "size": 3,
        "sort": [{"reviews.rating": {"order": "desc"}}],
        "highlight": {
          "fields": {"reviews.comment": {}}
        }
      }
    }
  }
}

// Nested aggregations
GET /products/_search
{
  "size": 0,
  "aggs": {
    "nested_reviews": {
      "nested": {
        "path": "reviews"
      },
      "aggs": {
        "avg_rating": {
          "avg": {"field": "reviews.rating"}
        },
        "rating_distribution": {
          "histogram": {
            "field": "reviews.rating",
            "interval": 1
          }
        },
        "back_to_products": {
          "reverse_nested": {},
          "aggs": {
            "avg_price": {
              "avg": {"field": "price"}
            }
          }
        }
      }
    }
  }
}

```

### Geo Queries

```json
// Geo distance query
GET /locations/_search
{
  "query": {
    "bool": {
      "filter": {
        "geo_distance": {
          "distance": "10km",
          "location": {
            "lat": 40.7128,
            "lon": -74.0060
          }
        }
      }
    }
  },
  "sort": [
    {
      "_geo_distance": {
        "location": {
          "lat": 40.7128,
          "lon": -74.0060
        },
        "order": "asc",
        "unit": "km"
      }
    }
  ]
}

// Geo bounding box
GET /locations/_search
{
  "query": {
    "geo_bounding_box": {
      "location": {
        "top_left": {
          "lat": 40.8,
          "lon": -74.1
        },
        "bottom_right": {
          "lat": 40.7,
          "lon": -73.9
        }
      }
    }
  }
}

// Geo aggregations
GET /locations/_search
{
  "size": 0,
  "aggs": {
    "location_grid": {
      "geohash_grid": {
        "field": "location",
        "precision": 5
      }
    },
    "distance_ranges": {
      "geo_distance": {
        "field": "location",
        "origin": "40.7128, -74.0060",
        "ranges": [
          {"to": 1000},
          {"from": 1000, "to": 5000},
          {"from": 5000}
        ]
      }
    }
  }
}

```

---

## 🔄 Data Processing Patterns

### Ingest Pipelines

```json
// Complete ingest pipeline
PUT /_ingest/pipeline/log_processing
{
  "description": "Process application logs",
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": [
          "%{TIMESTAMP_ISO8601:timestamp} \\[%{DATA:thread}\\] %{LOGLEVEL:level} %{DATA:logger} - %{GREEDYDATA:log_message}"
        ]
      }
    },
    {
      "date": {
        "field": "timestamp",
        "target_field": "@timestamp",
        "formats": ["ISO8601"]
      }
    },
    {
      "user_agent": {
        "field": "user_agent",
        "target_field": "ua"
      }
    },
    {
      "geoip": {
        "field": "client_ip",
        "target_field": "geo"
      }
    },
    {
      "script": {
        "source": "ctx.response_time_category = ctx.response_time < 100 ? 'fast' : ctx.response_time < 500 ? 'medium' : 'slow'"
      }
    },
    {
      "remove": {
        "field": ["timestamp", "message"]
      }
    }
  ],
  "on_failure": [
    {
      "set": {
        "field": "error.message",
        "value": "Failed to process log entry: {{ _ingest.on_failure_message }}"
      }
    }
  ]
}

// Use pipeline during indexing
POST /logs/_doc?pipeline=log_processing
{
  "message": "2023-10-15T14:30:15.123Z [http-nio-8080-exec-1] INFO  c.e.Application - Processing request",
  "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
  "client_ip": "192.168.1.100",
  "response_time": 250
}

```

### Runtime Fields

```json
// Runtime field in mapping
PUT /sales/_mapping
{
  "runtime": {
    "profit_margin": {
      "type": "double",
      "script": {
        "source": """
          if (doc['revenue'].size() != 0 && doc['cost'].size() != 0) {
            emit((doc['revenue'].value - doc['cost'].value) / doc['revenue'].value * 100);
          }
        """
      }
    },
    "quarter": {
      "type": "keyword",
      "script": {
        "source": """
          if (doc['@timestamp'].size() != 0) {
            def month = doc['@timestamp'].value.getMonthValue();
            if (month <= 3) emit('Q1');
            else if (month <= 6) emit('Q2');
            else if (month <= 9) emit('Q3');
            else emit('Q4');
          }
        """
      }
    }
  }
}

// Use runtime fields in queries
GET /sales/_search
{
  "query": {
    "range": {
      "profit_margin": {
        "gte": 20
      }
    }
  },
  "aggs": {
    "quarterly_sales": {
      "terms": {
        "field": "quarter"
      },
      "aggs": {
        "avg_margin": {
          "avg": {
            "field": "profit_margin"
          }
        }
      }
    }
  }
}

```

---

## 🚀 Performance Optimization Patterns

### Query Optimization

```json
// Optimized multi-field search
GET /products/_search
{
  "query": {
    "bool": {
      "must": {
        "multi_match": {
          "query": "wireless headphones",
          "fields": ["name^3", "description"],
          "type": "cross_fields",
          "operator": "and"
        }
      },
      "filter": [
        {"term": {"status": "active"}},
        {"range": {"price": {"gte": 50, "lte": 300}}},
        {"exists": {"field": "image_url"}}
      ],
      "should": [
        {"term": {"featured": true}},
        {"range": {"rating": {"gte": 4.0}}}
      ]
    }
  },
  "_source": ["name", "price", "rating", "image_url"],
  "size": 20,
  "from": 0,
  "sort": [
    {"_score": {"order": "desc"}},
    {"rating": {"order": "desc"}}
  ]
}

// Search after for deep pagination
GET /products/_search
{
  "query": {"match_all": {}},
  "sort": [
    {"price": {"order": "asc"}},
    {"_id": {"order": "asc"}}
  ],
  "size": 20,
  "search_after": [299.99, "prod_12345"]
}

```

### Index Settings Optimization

```json
// Performance-optimized index settings
PUT /high_volume_logs
{
  "settings": {
    "number_of_shards": 6,
    "number_of_replicas": 1,
    "refresh_interval": "30s",
    "index.translog.flush_threshold_size": "1gb",
    "index.translog.sync_interval": "30s",
    "index.codec": "best_compression",
    "index.compound_format": false,
    "index.max_result_window": 50000,
    "index.queries.cache.enabled": true,
    "index.requests.cache.enable": true
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "@timestamp": {"type": "date"},
      "level": {"type": "keyword", "doc_values": false},
      "message": {
        "type": "text",
        "norms": false,
        "index_options": "freqs"
      },
      "numeric_field": {
        "type": "scaled_float",
        "scaling_factor": 100
      }
    }
  }
}

```

---

## 💡 Quick Copy-Paste Templates

### Essential Cluster Operations

```bash
# Cluster health and info
GET /_cluster/health?level=indices
GET /_cat/nodes?v&h=name,node.role,heap.percent,cpu,load_1m
GET /_cat/indices?v&h=index,health,pri,rep,docs.count,store.size&s=store.size:desc

# Shard allocation
GET /_cat/shards?v&h=index,shard,prirep,state,docs,store,node&s=index,shard
GET /_cluster/allocation/explain

# Performance monitoring
GET /_nodes/stats/indices,os,jvm,process
GET /_nodes/hot_threads?threads=10
GET /_tasks?detailed=true&actions=*search*

# Index management
POST /index_name/_refresh
POST /index_name/_flush
POST /index_name/_forcemerge?max_num_segments=1
POST /index_name/_cache/clear

```

### Troubleshooting Commands

```bash
# Identify issues
GET /_cluster/health?level=shards
GET /_cat/recovery?v&h=index,shard,time,type,stage,source_node,target_node
GET /_cat/pending_tasks?v

# Memory and performance
GET /_nodes/stats/jvm?human
GET /_cat/thread_pool?v&h=name,active,queue,rejected&s=rejected:desc

# Storage and disk
GET /_cat/allocation?v&h=node,disk.used_percent,disk.avail&s=disk.used_percent:desc
GET /_nodes/stats/fs?human

# Query analysis
GET /index/_validate/query?explain=true
{
  "query": {"your_query_here"}
}

```

---