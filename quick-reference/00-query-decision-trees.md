# Query Decision Trees

## Systematic Approach to Query Construction

This guide provides decision trees and flowcharts to help you choose the right query type, aggregation, and optimization strategy for any search requirement. Use these trees to make informed decisions quickly and systematically.

---

## 🎯 How to Use These Decision Trees

### Purpose and Philosophy

Rather than memorizing dozens of query types, these decision trees help you:

- **Systematically evaluate** your search requirements
- **Choose optimal approaches** based on data and use case
- **Avoid common pitfalls** through guided decision-making
- **Learn patterns** that apply across different scenarios

### Decision Tree Structure

Each tree follows this pattern:

1. **Start with the business requirement** (what you want to achieve)
2. **Consider data characteristics** (how your data is structured)
3. **Evaluate performance constraints** (speed vs accuracy trade-offs)
4. **Choose implementation approach** (specific query types and settings)

---

## 🔍 Query Type Selection Decision Tree

### Primary Query Decision Flow

```
┌─────────────────────────────────────┐
│ What type of search do you need?    │
└─────────────────┬───────────────────┘
                  │
    ┌─────────────┼─────────────┐
    │             │             │
    ▼             ▼             ▼
┌───────┐    ┌─────────┐   ┌─────────┐
│ Exact │    │  Text   │   │ Complex │
│ Match │    │ Search  │   │ Logic   │
└───┬───┘    └────┬────┘   └────┬────┘
    │             │             │
    ▼             ▼             ▼

```

### Exact Match Path

```
┌─────────────────────────────────┐
│ EXACT MATCH REQUIRED            │
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│ What type of field?             │
└─────────────┬───────────────────┘
              │
    ┌─────────┼─────────┐
    │         │         │
    ▼         ▼         ▼
┌─────────┐ ┌─────┐ ┌─────────┐
│ Single  │ │Range│ │Multiple │
│ Value   │ │Query│ │ Values  │
└────┬────┘ └──┬──┘ └────┬────┘
     │         │         │
     ▼         ▼         ▼
┌─────────┐ ┌─────┐ ┌─────────┐
│  term   │ │range│ │ terms   │
└─────────┘ └─────┘ └─────────┘

```

**When to use each:**

| Query Type | Use When | Example |
| --- | --- | --- |
| **term** | Exact value in keyword field | `status: "published"` |
| **terms** | Match any of multiple exact values | `category: ["electronics", "books"]` |
| **range** | Numeric or date ranges | `price: 100-500`, `date: last 30 days` |
| **exists** | Field has any value | Documents with email field |
| **wildcard** | Pattern matching (use sparingly) | `filename: "*.pdf"` |

### Text Search Path

```
┌─────────────────────────────────┐
│ TEXT SEARCH REQUIRED            │
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│ How many fields to search?      │
└─────────────┬───────────────────┘
              │
        ┌─────┼─────┐
        │     │     │
        ▼     ▼     ▼
    ┌───────┐ ┌─────┐ ┌─────────┐
    │Single │ │Multi│ │All Text │
    │ Field │ │Field│ │ Fields  │
    └───┬───┘ └──┬──┘ └────┬────┘
        │        │         │
        ▼        ▼         ▼
    ┌───────┐ ┌─────────┐ ┌─────────┐
    │ match │ │multi_   │ │simple_  │
    │       │ │match    │ │query_   │
    │       │ │         │ │string   │
    └───────┘ └─────────┘ └─────────┘

```

**Text search decision criteria:**

```python
def choose_text_query():
    """Decision logic for text search queries"""

    decision_criteria = {
        'single_field_search': {
            'query_type': 'match',
            'use_when': 'Searching one specific field',
            'benefits': 'Simple, fast, good relevance scoring',
            'example': 'Search product titles only'
        },

        'multi_field_search': {
            'query_type': 'multi_match',
            'use_when': 'Searching across multiple specific fields',
            'benefits': 'Field boosting, cross-field scoring',
            'example': 'Search title^3, description^1, tags^2'
        },

        'simple_query_string': {
            'query_type': 'simple_query_string',
            'use_when': 'User-friendly search with operators',
            'benefits': 'Supports +, -, quotes, wildcards',
            'example': 'Google-like search box functionality'
        },

        'query_string': {
            'query_type': 'query_string',
            'use_when': 'Advanced users, complex syntax needed',
            'benefits': 'Full Lucene query syntax',
            'example': 'Power users, advanced search forms'
        }
    }

    return decision_criteria

```

### Complex Logic Path

```
┌─────────────────────────────────┐
│ COMPLEX LOGIC REQUIRED          │
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│ What type of logic?             │
└─────────────┬───────────────────┘
              │
    ┌─────────┼─────────┐
    │         │         │
    ▼         ▼         ▼
┌─────────┐ ┌─────┐ ┌─────────┐
│Boolean  │ │Score│ │Nested/  │
│Logic    │ │Boost│ │Parent   │
└────┬────┘ └──┬──┘ └────┬────┘
     │         │         │
     ▼         ▼         ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│ bool    │ │function_│ │ nested  │
│         │ │score    │ │ has_    │
│         │ │         │ │child/   │
│         │ │         │ │parent   │
└─────────┘ └─────────┘ └─────────┘

```

---

## 📊 Aggregation Selection Decision Tree

### Primary Aggregation Decision Flow

```
┌─────────────────────────────────────┐
│ What do you want to calculate?      │
└─────────────┬───────────────────────┘
              │
    ┌─────────┼─────────┐
    │         │         │
    ▼         ▼         ▼
┌─────────┐ ┌─────┐ ┌─────────┐
│Metrics  │ │Group│ │ Complex │
│(Stats)  │ │Data │ │Pipeline │
└────┬────┘ └──┬──┘ └────┬────┘
     │         │         │
     ▼         ▼         ▼

```

### Metrics Aggregations Decision

```python
def choose_metrics_aggregation():
    """Decision matrix for metrics aggregations"""

    metrics_decision = {
        'simple_calculations': {
            'sum': 'Total of all values',
            'avg': 'Average value',
            'min': 'Minimum value',
            'max': 'Maximum value',
            'value_count': 'Count of documents with field'
        },

        'statistical_analysis': {
            'stats': 'All basic stats (min, max, avg, sum, count)',
            'extended_stats': 'Additional variance, std_dev, bounds',
            'percentiles': 'Specific percentiles (50th, 95th, 99th)',
            'percentile_ranks': 'What percentile is specific value'
        },

        'unique_counting': {
            'cardinality': 'Approximate count of unique values',
            'use_when': 'Large datasets, approximate ok',
            'precision_threshold': 'Configure accuracy vs memory trade-off'
        },

        'geographic_metrics': {
            'geo_bounds': 'Bounding box of all geo points',
            'geo_centroid': 'Centroid of all geo points'
        }
    }

    return metrics_decision

```

### Bucket Aggregations Decision

```
┌─────────────────────────────────┐
│ GROUP BY WHAT TYPE OF FIELD?    │
└─────────────┬───────────────────┘
              │
    ┌─────────┼─────────┐
    │         │         │
    ▼         ▼         ▼
┌─────────┐ ┌─────┐ ┌─────────┐
│Keyword/ │ │Date/│ │Numeric  │
│Text     │ │Time │ │ Range   │
└────┬────┘ └──┬──┘ └────┬────┘
     │         │         │
     ▼         ▼         ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│ terms   │ │date_    │ │histogram│
│         │ │histogram│ │ range   │
└─────────┘ └─────────┘ └─────────┘

```

**Bucket aggregation selection guide:**

| Field Type | Aggregation | Use Case | Example |
| --- | --- | --- | --- |
| **Keyword** | terms | Group by categorical values | Categories, brands, users |
| **Date** | date_histogram | Time-based grouping | Daily/monthly trends |
| **Numeric** | histogram | Numeric ranges | Price ranges, age groups |
| **Geographic** | geo_distance | Distance-based grouping | "Stores within 5km" |
| **Boolean** | filters | Predefined groups | "Active vs inactive users" |

### Complex Pipeline Aggregations

```python
def choose_pipeline_aggregation():
    """When and how to use pipeline aggregations"""

    pipeline_patterns = {
        'bucket_calculations': {
            'bucket_script': 'Calculate values across bucket metrics',
            'bucket_selector': 'Filter buckets based on criteria',
            'bucket_sort': 'Sort and limit buckets',
            'example': 'Calculate conversion rate per category'
        },

        'time_series_analysis': {
            'moving_avg': 'Smooth out fluctuations',
            'derivative': 'Rate of change over time',
            'cumulative_sum': 'Running total',
            'example': 'Monthly growth rate calculation'
        },

        'statistical_analysis': {
            'stats_bucket': 'Statistics across buckets',
            'percentiles_bucket': 'Percentiles of bucket values',
            'max_bucket': 'Find bucket with highest value',
            'example': 'Find day with highest sales'
        }
    }

    return pipeline_patterns

```

---

## ⚡ Performance Optimization Decision Tree

### Query Performance Decision Flow

```
┌─────────────────────────────────┐
│ Is your query too slow?         │
└─────────────┬───────────────────┘
              │ YES
              ▼
┌─────────────────────────────────┐
│ What's the bottleneck?          │
└─────────────┬───────────────────┘
              │
    ┌─────────┼─────────┐
    │         │         │
    ▼         ▼         ▼
┌─────────┐ ┌─────┐ ┌─────────┐
│ Query   │ │Data │ │Cluster  │
│Construct│ │Size │ │Resource │
└────┬────┘ └──┬──┘ └────┬────┘
     │         │         │
     ▼         ▼         ▼

```

### Query Construction Optimization

```python
def optimize_query_construction():
    """Query optimization decision tree"""

    optimization_strategies = {
        'context_optimization': {
            'problem': 'Using query context for filtering',
            'solution': 'Move filters to filter context',
            'benefit': 'No scoring overhead, better caching',
            'example': 'date ranges, status filters'
        },

        'query_type_optimization': {
            'problem': 'Wrong query type for use case',
            'solution': 'Choose appropriate query type',
            'examples': {
                'exact_match': 'Use term instead of match',
                'prefix_search': 'Use prefix instead of wildcard',
                'fuzzy_search': 'Use fuzzy with limited edit distance'
            }
        },

        'aggregation_optimization': {
            'problem': 'Expensive aggregations',
            'solutions': {
                'reduce_cardinality': 'Limit size parameter',
                'use_composite': 'For large result sets',
                'avoid_deep_nesting': 'Limit aggregation depth',
                'filter_first': 'Reduce dataset before aggregating'
            }
        }
    }

    return optimization_strategies

```

### Data Size Optimization

```
┌─────────────────────────────────┐
│ Too much data to process?       │
└─────────────┬───────────────────┘
              │ YES
              ▼
┌─────────────────────────────────┐
│ Where can you reduce scope?     │
└─────────────┬───────────────────┘
              │
    ┌─────────┼─────────┐
    │         │         │
    ▼         ▼         ▼
┌─────────┐ ┌─────┐ ┌─────────┐
│ Time    │ │Field│ │Document │
│ Range   │ │Limit│ │ Filter  │
└────┬────┘ └──┬──┘ └────┬────┘
     │         │         │
     ▼         ▼         ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│Add date │ │_source  │ │ Better  │
│filters  │ │includes/│ │ query   │
│         │ │excludes │ │ filters │
└─────────┘ └─────────┘ └─────────┘

```

### Resource Optimization

```python
def optimize_cluster_resources():
    """Cluster-level optimization decisions"""

    resource_optimization = {
        'memory_optimization': {
            'heap_pressure': {
                'symptoms': 'High GC, slow queries',
                'solutions': [
                    'Reduce aggregation cardinality',
                    'Use doc_values instead of fielddata',
                    'Implement result pagination',
                    'Scale horizontally'
                ]
            },
            'cache_optimization': {
                'query_cache': 'Enable for repeated filter queries',
                'request_cache': 'Enable for aggregation-heavy workloads',
                'fielddata_cache': 'Monitor and limit usage'
            }
        },

        'cpu_optimization': {
            'query_complexity': 'Simplify expensive operations',
            'indexing_load': 'Balance indexing vs search workload',
            'merge_operations': 'Tune merge policy for workload'
        },

        'io_optimization': {
            'storage_tier': 'Use appropriate storage for data temperature',
            'replica_strategy': 'Balance availability vs resource usage',
            'refresh_interval': 'Tune for indexing throughput'
        }
    }

    return resource_optimization

```

---

## 🔧 Index Design Decision Tree

### Index Structure Decision Flow

```
┌─────────────────────────────────┐
│ How is your data accessed?      │
└─────────────┬───────────────────┘
              │
    ┌─────────┼─────────┐
    │         │         │
    ▼         ▼         ▼
┌─────────┐ ┌─────┐ ┌─────────┐
│Time-    │ │Mixed│ │Category-│
│Series   │ │Usage│ │Based    │
└────┬────┘ └──┬──┘ └────┬────┘
     │         │         │
     ▼         ▼         ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│Daily/   │ │Single   │ │Multiple │
│Monthly  │ │Large    │ │Smaller  │
│Indices  │ │Index    │ │Indices  │
└─────────┘ └─────────┘ └─────────┘

```

### Mapping Strategy Decision

```python
def choose_mapping_strategy():
    """Mapping design decision matrix"""

    mapping_decisions = {
        'field_type_selection': {
            'text_vs_keyword': {
                'text': 'Full-text search, analysis needed',
                'keyword': 'Exact match, aggregations, sorting',
                'both': 'Use multi-fields for both use cases'
            },
            'numeric_types': {
                'integer': 'Whole numbers within range',
                'long': 'Large whole numbers',
                'float': 'Decimal numbers, less precision',
                'double': 'Decimal numbers, high precision',
                'scaled_float': 'Fixed decimal places (price, percentage)'
            },
            'date_handling': {
                'date': 'Standard date field',
                'date_nanos': 'Nanosecond precision needed',
                'format': 'Specify expected date formats'
            }
        },

        'analysis_configuration': {
            'standard_analyzer': 'General text processing',
            'keyword_analyzer': 'No analysis, exact matching',
            'language_analyzer': 'Language-specific stemming',
            'custom_analyzer': 'Specific business requirements'
        },

        'performance_settings': {
            'doc_values': 'Enable for sorting/aggregations (default on)',
            'store': 'Enable only if retrieving original field values',
            'index': 'Disable for fields never searched',
            'norms': 'Disable for fields not needing scoring'
        }
    }

    return mapping_decisions

```

---

## 💡 Quick Decision Reference

### Common Query Patterns

| Use Case | Quick Decision | Query Type |
| --- | --- | --- |
| **Find exact match** | Single value? | `term` |
| **Find multiple values** | OR logic? | `terms` |
| **Search text** | Single field? | `match` |
| **Search multiple fields** | Different weights? | `multi_match` |
| **Combine conditions** | AND/OR/NOT logic? | `bool` |
| **Range queries** | Numbers/dates? | `range` |
| **Check field exists** | Has any value? | `exists` |
| **Pattern matching** | Wildcards needed? | `wildcard` |

### Common Aggregation Patterns

| Analysis Goal | Quick Decision | Aggregation Type |
| --- | --- | --- |
| **Count by category** | Group and count? | `terms` |
| **Time-based trends** | Group by time? | `date_histogram` |
| **Calculate average** | Single metric? | `avg` |
| **Get percentiles** | Distribution analysis? | `percentiles` |
| **Count unique values** | Approximate ok? | `cardinality` |
| **Geographic analysis** | Location-based? | `geo_distance` |

### Performance Optimization Quick Checks

| Symptom | Quick Check | Likely Solution |
| --- | --- | --- |
| **Slow queries** | Using filter context? | Move filters from `must` to `filter` |
| **High memory** | Large aggregations? | Reduce `size` parameter |
| **Long indexing** | Real-time needed? | Increase `refresh_interval` |
| **Storage issues** | Old data needed? | Implement ILM policy |

---