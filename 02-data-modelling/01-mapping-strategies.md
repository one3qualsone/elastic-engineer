# Mapping Strategies

## Configuring Elasticsearch for Optimal Performance and Search Quality

Mappings define how Elasticsearch stores and indexes your documents. Getting mappings right is crucial—they determine what queries are possible, how fast they execute, and how much storage you need. Unlike databases where you can often change schema later, mapping decisions have long-term consequences and are expensive to change.

---

## 🗺️ Understanding Mappings

### What Mappings Control

**Mappings determine:**

- How each field is stored and indexed
- What data types are used for each field
- How text is analyzed for search
- What operations are possible on each field
- How much storage and memory each field requires

**The mapping impact:**

```json
// Same document, different mappings = different capabilities

// Mapping 1: Basic text search
"description": {"type": "text"}

// Mapping 2: Advanced search with exact matching
"description": {
  "type": "text",
  "analyzer": "english",
  "fields": {
    "keyword": {"type": "keyword"},
    "search": {"type": "text", "analyzer": "search_analyzer"}
  }
}

```

### Dynamic vs Explicit Mappings

**Dynamic mapping (Elasticsearch guesses):**

- Convenient for rapid prototyping
- Often guesses wrong for production use
- Can lead to mapping conflicts
- Difficult to optimise later

**Explicit mapping (you define):**

- Full control over field behavior
- Optimised for your specific use case
- Consistent across environments
- Easier to troubleshoot and tune

**Best practice:** Start with explicit mappings for production systems.

---

## 📝 Field Type Decision Framework

### The Core Field Types

| Field Type | Use Cases | Storage Characteristics | Query Capabilities |
| --- | --- | --- | --- |
| **text** | Full-text search, content analysis | Analyzed and stored | match, match_phrase, fuzzy queries |
| **keyword** | Exact matching, filtering, aggregations | Not analyzed, stored as-is | term, terms, exists queries |
| **date** | Timestamps, date ranges | Stored as milliseconds | range queries, date math |
| **numeric** | Counts, prices, scores | Binary encoding | range, term queries, math aggregations |
| **boolean** | True/false flags | Single bit | term queries |
| **object** | Nested structures | Flattened key-value pairs | Dot notation queries |
| **nested** | Array of objects with relationships | Separate hidden documents | nested queries |

### Text vs Keyword: The Critical Decision

**When to use text:**

```json
{
  "product_description": {
    "type": "text",
    "analyzer": "english"
  }
}

```

- Full-text search required
- Content that benefits from linguistic analysis
- User-generated content with varied vocabulary
- Need relevance scoring

**When to use keyword:**

```json
{
  "product_category": {
    "type": "keyword"
  }
}

```

- Exact matching requirements
- Filtering and aggregations
- IDs, status codes, categories
- Need sorting capability

**The multi-field approach:**

```json
{
  "product_name": {
    "type": "text",
    "analyzer": "standard",
    "fields": {
      "keyword": {
        "type": "keyword",
        "ignore_above": 256
      },
      "search": {
        "type": "text",
        "analyzer": "english"
      }
    }
  }
}

```

**Benefits:**

- `product_name`: Full-text search with standard analysis
- `product_name.keyword`: Exact matching and aggregations
- `product_name.search`: Enhanced search with stemming and stop words

---

## 🔬 Text Analysis: Making Search Intelligent

### The Analysis Process

**Analysis pipeline:**

```
Input Text → Character Filters → Tokenizer → Token Filters → Output Tokens

```

**Example:** "The quick brown FOX jumps!"

```
1. Input: "The quick brown FOX jumps!"
2. Character filters: "The quick brown FOX jumps!" (lowercase filter)
3. Tokenizer: ["The", "quick", "brown", "FOX", "jumps!"] (standard tokenizer)
4. Token filters: ["quick", "brown", "fox", "jump"] (lowercase + stop words + stemming)
5. Output: Indexed tokens for search

```

### Built-in Analyzers

| Analyzer | Best For | Characteristics |
| --- | --- | --- |
| **standard** | General purpose text | Lowercase, basic tokenization |
| **english** | English content | Stemming, stop words, possessives |
| **keyword** | Exact matching | No analysis, stored as-is |
| **simple** | Basic text | Lowercase, letter tokenization |
| **whitespace** | Code, identifiers | Split on whitespace only |

**Practical example:**

```json
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "english",
        "fields": {
          "raw": {"type": "keyword"},
          "simple": {"type": "text", "analyzer": "simple"}
        }
      }
    }
  }
}

```

### Custom Analyzers for Specific Needs

**E-commerce search analyzer:**

```json
{
  "settings": {
    "analysis": {
      "char_filter": {
        "product_char_filter": {
          "type": "pattern_replace",
          "pattern": "\\s+",
          "replacement": " "
        }
      },
      "tokenizer": {
        "product_tokenizer": {
          "type": "pattern",
          "pattern": "[\\W+]"
        }
      },
      "filter": {
        "product_stemmer": {
          "type": "stemmer",
          "language": "english"
        },
        "product_synonyms": {
          "type": "synonym",
          "synonyms": [
            "laptop,notebook,computer",
            "mobile,phone,smartphone"
          ]
        }
      },
      "analyzer": {
        "product_search": {
          "char_filter": ["product_char_filter"],
          "tokenizer": "product_tokenizer",
          "filter": [
            "lowercase",
            "product_synonyms",
            "product_stemmer"
          ]
        }
      }
    }
  }
}

```

### Domain-Specific Analysis Examples

**Log analysis mapping:**

```json
{
  "log_message": {
    "type": "text",
    "analyzer": "keyword",
    "fields": {
      "search": {
        "type": "text",
        "analyzer": "simple"
      },
      "structured": {
        "type": "text",
        "analyzer": "log_analyzer"
      }
    }
  }
}

```

**Email/contact analysis:**

```json
{
  "email": {
    "type": "keyword",
    "fields": {
      "domain": {
        "type": "text",
        "analyzer": "email_domain_analyzer"
      }
    }
  }
}

```

---

## 📊 Numeric and Date Field Optimization

### Numeric Field Types

**Choose the right numeric type:**

| Type | Range | Use Case | Storage |
| --- | --- | --- | --- |
| **byte** | -128 to 127 | Small counters, flags | 1 byte |
| **short** | -32,768 to 32,767 | Medium counters | 2 bytes |
| **integer** | -2³¹ to 2³¹-1 | Standard integers | 4 bytes |
| **long** | -2⁶³ to 2⁶³-1 | Large numbers, timestamps | 8 bytes |
| **float** | 32-bit IEEE 754 | Approximate decimals | 4 bytes |
| **double** | 64-bit IEEE 754 | Precise decimals | 8 bytes |
| **scaled_float** | Long with scaling factor | Prices, percentages | 8 bytes + factor |

**Practical examples:**

```json
{
  "product_price": {
    "type": "scaled_float",
    "scaling_factor": 100
  },
  "view_count": {
    "type": "integer"
  },
  "rating": {
    "type": "float"
  },
  "timestamp_ms": {
    "type": "long"
  }
}

```

### Date Field Configuration

**Date formats and parsing:**

```json
{
  "created_at": {
    "type": "date",
    "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
  },
  "publish_date": {
    "type": "date",
    "format": "strict_date_optional_time"
  }
}

```

**Common date patterns:**

- `yyyy-MM-dd`: "2024-01-15"
- `yyyy-MM-dd HH:mm:ss`: "2024-01-15 10:30:00"
- `strict_date_optional_time`: ISO 8601 format
- `epoch_millis`: Unix timestamp in milliseconds

---

## 🗂️ Complex Data Structures

### Object vs Nested Fields

**Object fields (default):**

```json
{
  "user": {
    "properties": {
      "name": {"type": "text"},
      "email": {"type": "keyword"}
    }
  }
}

```

**Flattened structure in Elasticsearch:**

```json
{
  "user.name": "John Doe",
  "user.email": "john@example.com"
}

```

**Nested fields (for arrays of objects):**

```json
{
  "comments": {
    "type": "nested",
    "properties": {
      "author": {"type": "keyword"},
      "text": {"type": "text"},
      "rating": {"type": "integer"}
    }
  }
}

```

**When to use nested:**

| Scenario | Object | Nested |
| --- | --- | --- |
| **Single object** | ✅ | ❌ (unnecessary overhead) |
| **Array of objects** | ❌ (loses relationships) | ✅ |
| **Simple key-value pairs** | ✅ | ❌ |
| **Complex array queries** | ❌ | ✅ |

**Example problem with object arrays:**

```json
// Document
{
  "products": [
    {"name": "Laptop", "price": 1000, "category": "Electronics"},
    {"name": "Book", "price": 20, "category": "Education"}
  ]
}

// This query incorrectly matches (Laptop + Education)
{
  "query": {
    "bool": {
      "must": [
        {"term": {"products.category": "Education"}},
        {"range": {"products.price": {"gte": 500}}}
      ]
    }
  }
}

```

**Nested solution:**

```json
{
  "query": {
    "nested": {
      "path": "products",
      "query": {
        "bool": {
          "must": [
            {"term": {"products.category": "Education"}},
            {"range": {"products.price": {"gte": 500}}}
          ]
        }
      }
    }
  }
}

```

---

## ⚙️ Index Settings and Performance

### Essential Index Settings

**Performance-critical settings:**

```json
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "1s",
    "max_result_window": 10000,
    "analysis": {
      // Custom analyzers
    }
  }
}

```

### Shard Strategy

**Shard sizing guidelines:**

- Target shard size: 10-50GB for most workloads
- Maximum recommended: 200GB per shard
- Minimum practical: 1GB per shard

**Calculation example:**

```
Expected data: 300GB
Target shard size: 30GB
Recommended shards: 300GB / 30GB = 10 shards

Growth factor: 2x over 2 years
Future data: 600GB
Shards needed: 600GB / 30GB = 20 shards

Final decision: 20 primary shards

```

### Refresh Interval Optimization

**Refresh interval impact:**

| Interval | Use Case | Trade-off |
| --- | --- | --- |
| **1s** (default) | Real-time search | Higher resource usage |
| **30s** | Near real-time acceptable | Better indexing performance |
| **-1** (disabled) | Bulk loading | Maximum indexing speed |
| **Manual** | Controlled visibility | Explicit control over freshness |

**Dynamic adjustment:**

```json
// During bulk loading
PUT /my-index/_settings
{
  "refresh_interval": -1,
  "number_of_replicas": 0
}

// After bulk loading
PUT /my-index/_settings
{
  "refresh_interval": "1s",
  "number_of_replicas": 1
}

```

---

## 🎯 Mapping Patterns by Use Case

### E-commerce Product Catalog

```json
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "product_search": {
          "tokenizer": "standard",
          "filter": ["lowercase", "synonym_filter"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "id": {"type": "keyword"},
      "name": {
        "type": "text",
        "analyzer": "product_search",
        "fields": {
          "keyword": {"type": "keyword"},
          "autocomplete": {
            "type": "search_as_you_type"
          }
        }
      },
      "description": {
        "type": "text",
        "analyzer": "english"
      },
      "price": {
        "type": "scaled_float",
        "scaling_factor": 100
      },
      "categories": {
        "type": "keyword"
      },
      "attributes": {
        "type": "nested",
        "properties": {
          "name": {"type": "keyword"},
          "value": {"type": "keyword"}
        }
      },
      "availability": {
        "properties": {
          "in_stock": {"type": "boolean"},
          "quantity": {"type": "integer"}
        }
      },
      "reviews": {
        "properties": {
          "average_rating": {"type": "float"},
          "count": {"type": "integer"}
        }
      }
    }
  }
}

```

### Application Logs

```json
{
  "settings": {
    "number_of_shards": 3,
    "refresh_interval": "5s",
    "index": {
      "lifecycle": {
        "name": "logs_policy",
        "rollover_alias": "logs"
      }
    }
  },
  "mappings": {
    "properties": {
      "@timestamp": {
        "type": "date",
        "format": "strict_date_optional_time||epoch_millis"
      },
      "level": {"type": "keyword"},
      "message": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 1024
          }
        }
      },
      "service": {
        "properties": {
          "name": {"type": "keyword"},
          "version": {"type": "keyword"},
          "environment": {"type": "keyword"}
        }
      },
      "host": {
        "properties": {
          "name": {"type": "keyword"},
          "ip": {"type": "ip"}
        }
      },
      "error": {
        "properties": {
          "type": {"type": "keyword"},
          "message": {"type": "text"},
          "stack_trace": {
            "type": "text",
            "index": false
          }
        }
      }
    }
  }
}

```

### User Analytics Events

```json
{
  "mappings": {
    "properties": {
      "user_id": {"type": "keyword"},
      "session_id": {"type": "keyword"},
      "timestamp": {"type": "date"},
      "event_type": {"type": "keyword"},
      "page": {
        "properties": {
          "url": {"type": "keyword"},
          "title": {"type": "text"},
          "category": {"type": "keyword"}
        }
      },
      "user_agent": {
        "properties": {
          "browser": {"type": "keyword"},
          "os": {"type": "keyword"},
          "device": {"type": "keyword"}
        }
      },
      "geo": {
        "properties": {
          "country": {"type": "keyword"},
          "city": {"type": "keyword"},
          "location": {"type": "geo_point"}
        }
      },
      "custom_properties": {
        "type": "flattened"
      }
    }
  }
}

```

---

## 🚫 Common Mapping Mistakes

### Mistake 1: Over-Indexing

**❌ Problem:**

```json
{
  "user_data": {
    "type": "text"  // Everything searchable
  }
}

```

**✅ Solution:**

```json
{
  "user_data": {
    "type": "object",
    "properties": {
      "id": {"type": "keyword"},
      "name": {"type": "text"},
      "internal_notes": {
        "type": "text",
        "index": false  // Store but don't index
      }
    }
  }
}

```

### Mistake 2: Wrong Field Types

**❌ Problem:**

```json
{
  "product_id": {"type": "text"},  // Should be keyword
  "description": {"type": "keyword"}  // Should be text
}

```

**✅ Solution:**

```json
{
  "product_id": {"type": "keyword"},
  "description": {"type": "text", "analyzer": "english"}
}

```

### Mistake 3: Ignoring Doc Values

**❌ Problem:**

```json
{
  "large_text_field": {
    "type": "keyword"  // Will use lots of memory for aggregations
  }
}

```

**✅ Solution:**

```json
{
  "large_text_field": {
    "type": "keyword",
    "doc_values": false,  // Disable if no aggregations needed
    "fields": {
      "agg": {
        "type": "keyword",
        "ignore_above": 100  // Separate field for aggregations
      }
    }
  }
}

```

---

## 📈 Performance Optimization Strategies

### Memory Usage Optimization

**Field data vs doc values:**

- **Doc values:** Disk-based, efficient for sorting/aggregations
- **Field data:** Memory-based, fast but memory-intensive

**Optimization techniques:**

```json
{
  "high_cardinality_field": {
    "type": "keyword",
    "doc_values": true,      // Enable for aggregations
    "index": false,          // Disable if no term queries
    "ignore_above": 256      // Limit field length
  },
  "full_text_no_agg": {
    "type": "text",
    "fielddata": false       // Disable fielddata
  }
}

```

### Storage Optimization

**Best practices:**

```json
{
  "_source": {
    "enabled": true,
    "excludes": ["large_binary_field", "debug_info"]
  },
  "properties": {
    "stored_only_field": {
      "type": "keyword",
      "index": false,
      "doc_values": false
    },
    "compressed_field": {
      "type": "binary"
    }
  }
}

```

---

## 💡 Key Takeaways

✅ **Explicit mappings prevent surprises and enable optimization**

✅ **Text vs keyword choice depends on search vs exact-match requirements**

✅ **Custom analyzers improve search quality for specific domains**

✅ **Nested fields solve array-of-objects relationship problems**

✅ **Index settings significantly impact performance and resource usage**

✅ **Different use cases require different mapping strategies**

✅ **Avoiding common mistakes saves time and performance issues**

---