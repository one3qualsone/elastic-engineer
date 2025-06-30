# Schema Evolution Patterns

## Managing Change in Production Elasticsearch Systems

Schema evolution in Elasticsearch requires careful planning because many mapping changes are impossible after index creation. Unlike traditional databases where you can `ALTER TABLE`, Elasticsearch mappings are largely immutable. This constraint forces you to think strategically about change management and adopt patterns that enable safe evolution over time.

---

## 🔄 The Schema Evolution Challenge

### Why Elasticsearch Mappings Are Different

**Traditional database approach:**

```sql
-- Easy schema changes
ALTER TABLE products ADD COLUMN new_field VARCHAR(255);
ALTER TABLE products MODIFY COLUMN price DECIMAL(10,2);

```

**Elasticsearch reality:**

```json
// Most mapping changes require reindexing
PUT /products/_mapping
{
  "properties": {
    "new_field": {"type": "text"}  // ✅ Adding fields works
  }
}

// ❌ These changes are impossible:
// - Change field type (text → keyword)
// - Change analyzer
// - Change index/doc_values settings
// - Remove fields (they remain in mapping)

```

### The Cost of Getting It Wrong

**Symptoms of poor schema planning:**

- Frequent reindexing operations (downtime and resource usage)
- Degraded search quality due to wrong field types
- Complex application code to handle multiple schema versions
- Performance issues from suboptimal mappings
- Inability to add required features without major migrations

**Real-world example:**

```json
// Initial mapping (mistake)
{
  "product_category": {"type": "text"}  // Should be keyword for filters
}

// Application grows, needs category filtering
// Now requires full reindex of production data
// Estimated downtime: 6 hours for 100M documents

```

---

## 🏗️ Evolution-Friendly Design Patterns

### Pattern 1: Multi-Field Strategy

**Design for multiple access patterns upfront:**

```json
{
  "product_name": {
    "type": "text",
    "analyzer": "standard",
    "fields": {
      "keyword": {"type": "keyword"},
      "search": {"type": "text", "analyzer": "english"},
      "autocomplete": {"type": "search_as_you_type"},
      "sort": {"type": "keyword", "normalizer": "lowercase"}
    }
  }
}

```

**Benefits:**

- Supports various query types without reindexing
- Easy to optimize for new use cases
- Minimal additional storage cost
- Future-proof for unknown requirements

### Pattern 2: Extensible Metadata Objects

**Design flexible containers for evolving requirements:**

```json
{
  "product_id": "PROD-123",
  "name": "Laptop",

  // Core fields (stable)
  "price": 999.99,
  "category": "Electronics",

  // Extensible metadata
  "metadata": {
    "vendor_specific": {
      "warranty_years": 3,
      "energy_rating": "A+"
    },
    "experimental": {
      "ai_generated_tags": ["portable", "business"]
    }
  },

  // Dynamic attributes for varying product types
  "attributes": {
    "screen_size": "15 inch",
    "processor": "Intel i7",
    "ram": "16GB"
  }
}

```

### Pattern 3: Version-Aware Documents

**Include schema version information:**

```json
{
  "document_version": "2.1",
  "schema_migration_date": "2024-01-15T10:00:00Z",

  // Legacy fields (maintained for compatibility)
  "old_category_format": "electronics-laptops",

  // New fields
  "categories": [
    {"level": 1, "name": "Electronics"},
    {"level": 2, "name": "Computers"},
    {"level": 3, "name": "Laptops"}
  ],

  // Migration status
  "migration_flags": {
    "category_migration_complete": true,
    "v2_search_fields_added": true
  }
}

```

---

## 🔄 Safe Schema Change Strategies

### Strategy 1: Additive Changes (Zero Downtime)

**Safe operations that don't require reindexing:**

**Adding new fields:**

```json
PUT /products/_mapping
{
  "properties": {
    "new_search_field": {
      "type": "text",
      "analyzer": "english"
    },
    "feature_flags": {
      "type": "object",
      "properties": {
        "enhanced_search": {"type": "boolean"}
      }
    }
  }
}

```

**Adding multi-fields to existing fields:**

```json
PUT /products/_mapping
{
  "properties": {
    "description": {
      "type": "text",
      "fields": {
        "length": {
          "type": "token_count",
          "analyzer": "standard"
        }
      }
    }
  }
}

```

### Strategy 2: Parallel Index Pattern

**For breaking changes, run old and new indices in parallel:**

**Phase 1: Create new index with improved mapping**

```json
PUT /products_v2
{
  "mappings": {
    "properties": {
      "category": {"type": "keyword"},  // Fixed: was text
      "description": {
        "type": "text",
        "analyzer": "enhanced_analyzer"  // Improved analyzer
      }
    }
  }
}

```

**Phase 2: Dual-write to both indices**

```python
def index_product(product_data):
    # Write to old index (backwards compatibility)
    es.index(index="products", body=product_data)

    # Write to new index (improved mapping)
    enhanced_data = transform_for_v2(product_data)
    es.index(index="products_v2", body=enhanced_data)

```

**Phase 3: Switch reads to new index**

```python
# Gradual migration with feature flags
def search_products(query):
    if feature_flag_enabled("use_v2_index"):
        return search_v2_index(query)
    else:
        return search_v1_index(query)

```

**Phase 4: Remove old index**

```json
DELETE /products
POST /_aliases
{
  "actions": [
    {"add": {"index": "products_v2", "alias": "products"}}
  ]
}

```

### Strategy 3: Rolling Reindex Pattern

**For large datasets that can't be dual-written:**

**Chunk-based reindexing:**

```json
POST /_reindex
{
  "source": {
    "index": "products_old",
    "size": 10000,
    "query": {
      "range": {
        "product_id": {
          "gte": "PROD-000000",
          "lt": "PROD-100000"
        }
      }
    }
  },
  "dest": {
    "index": "products_new"
  }
}

```

**Progress tracking:**

```json
GET /_tasks?detailed=true&actions=*/reindex

{
  "task_id": "node:12345",
  "status": {
    "total": 1000000,
    "updated": 750000,
    "batches": 75,
    "version_conflicts": 0
  }
}

```

---

## 📊 Data Migration Patterns

### Pattern 1: In-Place Data Transformation

**Use update by query for compatible changes:**

```json
POST /products/_update_by_query
{
  "script": {
    "source": """
      // Add computed field
      ctx._source.search_boost = ctx._source.view_count * 0.1;

      // Normalize existing field
      if (ctx._source.category != null) {
        ctx._source.category = ctx._source.category.toLowerCase();
      }

      // Add version tracking
      ctx._source.last_updated = params.update_time;
    """,
    "params": {
      "update_time": "2024-01-15T10:00:00Z"
    }
  }
}

```

### Pattern 2: ETL-Style Migration

**Complex transformations using external processing:**

```python
from elasticsearch import Elasticsearch
from elasticsearch.helpers import scan, bulk

def migrate_products():
    es = Elasticsearch()

    # Extract from old index
    documents = scan(
        es,
        index="products_old",
        query={"match_all": {}},
        scroll='5m'
    )

    # Transform documents
    def transform_doc(doc):
        source = doc['_source']

        # Complex category transformation
        old_category = source.get('category_id')
        source['categories'] = lookup_category_hierarchy(old_category)

        # Enhanced search fields
        source['search_text'] = create_search_text(source)

        # Remove deprecated fields
        source.pop('deprecated_field', None)

        return {
            '_index': 'products_new',
            '_id': doc['_id'],
            '_source': source
        }

    # Load to new index
    bulk(es, (transform_doc(doc) for doc in documents))

```

### Pattern 3: Event-Driven Migration

**Use change streams for real-time migration:**

```python
class ProductMigrationService:
    def __init__(self):
        self.es = Elasticsearch()
        self.kafka_consumer = KafkaConsumer('product-changes')

    def process_product_change(self, event):
        if event['type'] == 'UPDATE':
            # Transform and index to new schema
            transformed = self.transform_product(event['data'])
            self.es.index(
                index='products_v2',
                id=event['product_id'],
                body=transformed
            )

    def start_migration(self):
        for message in self.kafka_consumer:
            self.process_product_change(message.value)

```

---

## 🕐 Timing and Rollback Strategies

### Migration Timing Considerations

**Low-traffic windows:**

```python
import schedule
import time

def safe_migration_window():
    current_hour = datetime.now().hour
    # Run during low-traffic hours (2-6 AM UTC)
    return 2 <= current_hour <= 6

def run_migration_batch():
    if safe_migration_window():
        migrate_next_batch()
        time.sleep(300)  # 5-minute intervals

```

**Resource-aware migration:**

```python
def adaptive_migration():
    cluster_stats = es.cluster.stats()
    cpu_usage = cluster_stats['nodes']['process']['cpu']['percent']

    if cpu_usage < 70:
        batch_size = 10000  # Normal speed
    elif cpu_usage < 85:
        batch_size = 5000   # Slower
    else:
        time.sleep(60)      # Pause during high load
        return

    migrate_batch(batch_size)

```

### Rollback Strategies

**Alias-based rollback:**

```json
// Current state
GET /_cat/aliases/products
products products_v2 - - -

// Rollback to v1
POST /_aliases
{
  "actions": [
    {"remove": {"index": "products_v2", "alias": "products"}},
    {"add": {"index": "products_v1", "alias": "products"}}
  ]
}

```

**Snapshot-based rollback:**

```json
// Create snapshot before migration
PUT /_snapshot/backup_repo/pre_migration_snapshot
{
  "indices": "products",
  "ignore_unavailable": true,
  "include_global_state": false
}

// Rollback if needed
POST /_snapshot/backup_repo/pre_migration_snapshot/_restore
{
  "indices": "products",
  "rename_pattern": "products",
  "rename_replacement": "products_restored"
}

```

---

## 🛠️ Tools and Automation

### Migration Scripts and Tooling

**Reindex with transformation script:**

```json
POST /_reindex
{
  "source": {
    "index": "products_old"
  },
  "dest": {
    "index": "products_new"
  },
  "script": {
    "source": """
      // Version 2 schema transformation
      ctx._source.schema_version = '2.0';

      // Transform category from ID to object
      if (ctx._source.category_id != null) {
        ctx._source.category = [
          'id': ctx._source.category_id,
          'name': params.category_lookup[ctx._source.category_id]
        ];
        ctx._source.remove('category_id');
      }

      // Add computed search fields
      def searchTerms = [];
      if (ctx._source.name != null) searchTerms.add(ctx._source.name);
      if (ctx._source.brand != null) searchTerms.add(ctx._source.brand);
      ctx._source.search_all = String.join(' ', searchTerms);
    """,
    "params": {
      "category_lookup": {
        "1": "Electronics",
        "2": "Clothing",
        "3": "Books"
      }
    }
  }
}

```

### Validation and Testing

**Schema validation pipeline:**

```python
def validate_migration():
    """Comprehensive migration validation"""

    # 1. Document count validation
    old_count = es.count(index="products_old")['count']
    new_count = es.count(index="products_new")['count']
    assert abs(old_count - new_count) < 100, "Document count mismatch"

    # 2. Sample document validation
    sample_docs = es.search(
        index="products_old",
        size=100,
        body={"query": {"match_all": {}}}
    )

    for doc in sample_docs['hits']['hits']:
        old_doc = doc['_source']
        new_doc = es.get(index="products_new", id=doc['_id'])['_source']

        # Validate transformation logic
        assert validate_document_transformation(old_doc, new_doc)

    # 3. Search quality validation
    test_queries = load_test_queries()
    for query in test_queries:
        old_results = search_old_index(query)
        new_results = search_new_index(query)

        # Ensure search quality hasn't degraded
        assert compare_search_quality(old_results, new_results) > 0.95

```

---

## 📈 Advanced Evolution Patterns

### Pattern 1: Feature Flag-Driven Schema Changes

**Gradual schema adoption:**

```json
{
  "product_id": "PROD-123",

  // Old format (always present)
  "category": "electronics",

  // New format (conditionally used)
  "category_v2": {
    "hierarchy": ["Electronics", "Computers", "Laptops"],
    "tags": ["portable", "business"],
    "ml_classification": {
      "confidence": 0.95,
      "predicted_category": "business-laptop"
    }
  },

  // Feature flags control usage
  "feature_flags": {
    "use_v2_categories": true,
    "enable_ml_classification": false
  }
}

```

### Pattern 2: A/B Testing Schema Changes

**Compare schema performance:**

```python
def search_with_schema_test(query, user_id):
    # Consistent assignment based on user ID
    schema_version = "v2" if hash(user_id) % 100 < 10 else "v1"

    if schema_version == "v2":
        results = search_v2_schema(query)
        log_performance("schema_v2", query, results)
    else:
        results = search_v1_schema(query)
        log_performance("schema_v1", query, results)

    return results

```

### Pattern 3: Machine Learning-Driven Evolution

**Use ML to guide schema improvements:**

```python
def analyze_query_patterns():
    """Analyze search patterns to suggest schema improvements"""

    # Collect query analytics
    failed_queries = get_low_performance_queries()
    popular_filters = get_most_used_filters()

    suggestions = []

    # Suggest new keyword fields for popular filters
    for field, usage in popular_filters.items():
        if usage > 1000 and not has_keyword_mapping(field):
            suggestions.append({
                'type': 'add_keyword_field',
                'field': field,
                'usage': usage
            })

    # Suggest analyzer improvements for failed queries
    for query in failed_queries:
        if query['zero_results'] and contains_synonyms(query['text']):
            suggestions.append({
                'type': 'improve_analyzer',
                'query': query['text'],
                'suggested_synonyms': detect_synonyms(query['text'])
            })

    return suggestions

```

---

## ⚠️ Common Evolution Pitfalls

### Pitfall 1: Big Bang Migrations

**❌ Problem:**

```python
# Dangerous: All-at-once migration
def migrate_everything():
    reindex_all_data()      # Hours of downtime
    update_application()    # Risk of rollback issues
    hope_nothing_breaks()   # 🤞

```

**✅ Solution:**

```python
# Safe: Gradual migration
def migrate_gradually():
    create_new_index()
    start_dual_writes()
    migrate_in_batches()
    validate_each_batch()
    switch_reads_gradually()
    cleanup_old_index()

```

### Pitfall 2: Ignoring Application Compatibility

**❌ Problem:**

```json
// Remove field without updating application
{
  // "old_field": "removed", ← Application still expects this
  "new_field": "replacement_value"
}

```

**✅ Solution:**

```json
// Maintain compatibility during transition
{
  "old_field": "legacy_value",    // Keep until app updated
  "new_field": "replacement_value", // New applications use this
  "migration_status": "transitioning"
}

```

### Pitfall 3: Poor Rollback Planning

**❌ Problem:**

- No rollback strategy defined
- No way to revert to previous schema
- No validation of migration success

**✅ Solution:**

- Always create snapshots before major changes
- Plan rollback steps before starting migration
- Implement comprehensive validation
- Use aliases for easy index switching

---

## 💡 Key Takeaways

✅ **Plan for evolution from day one with flexible mapping patterns**

✅ **Most mapping changes require reindexing—plan accordingly**

✅ **Use parallel index patterns for safe breaking changes**

✅ **Implement comprehensive validation for all migrations**

✅ **Gradual migration reduces risk and enables quick rollback**

✅ **Feature flags enable safe testing of schema changes**

✅ **Always have a rollback plan before starting migrations**

---
