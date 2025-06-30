# Progressive Query Building Exercises

## From Simple Searches to Complex Analytics

These exercises build your query construction skills progressively, starting with basic searches and advancing to complex multi-stage analytics. Each exercise builds on previous concepts while introducing new techniques.

---

## 🎯 Exercise Philosophy

### Learning Through Progressive Complexity

Rather than jumping into complex queries, these exercises follow a **systematic progression** that mirrors real-world development:

1. **Start Simple** - Basic match queries and filters
2. **Add Complexity** - Combine multiple conditions
3. **Introduce Analytics** - Add aggregations and metrics
4. **Optimise Performance** - Improve query efficiency
5. **Scale Up** - Handle production-level complexity

### Exercise Structure

Each exercise includes:

- **Scenario** - Real business requirement
- **Starting Point** - Sample data and basic query
- **Progressive Steps** - Building complexity incrementally
- **Best Practices** - Why certain approaches work better
- **Extensions** - Further challenges to explore

---

## 📊 Exercise 1: E-Commerce Product Search

### Scenario: Building a Product Search Engine

You're building search functionality for an e-commerce platform. Start with basic product lookups and progress to sophisticated search with faceting, recommendations, and performance optimization.

### Sample Data Setup

```json
PUT /products
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "fields": {
          "keyword": {"type": "keyword"},
          "suggest": {"type": "completion"}
        }
      },
      "description": {"type": "text"},
      "category": {"type": "keyword"},
      "brand": {"type": "keyword"},
      "price": {"type": "float"},
      "rating": {"type": "float"},
      "review_count": {"type": "integer"},
      "in_stock": {"type": "boolean"},
      "tags": {"type": "keyword"},
      "release_date": {"type": "date"},
      "specifications": {
        "type": "nested",
        "properties": {
          "name": {"type": "keyword"},
          "value": {"type": "keyword"}
        }
      }
    }
  }
}

# Sample documents
POST /products/_bulk
{"index": {}}
{"name": "MacBook Pro 16-inch", "description": "Powerful laptop for professionals", "category": "Electronics", "brand": "Apple", "price": 2399.99, "rating": 4.5, "review_count": 127, "in_stock": true, "tags": ["laptop", "professional", "high-performance"], "release_date": "2023-01-15", "specifications": [{"name": "RAM", "value": "16GB"}, {"name": "Storage", "value": "512GB"}]}
{"index": {}}
{"name": "Dell XPS 13", "description": "Ultrabook for everyday computing", "category": "Electronics", "brand": "Dell", "price": 1299.99, "rating": 4.2, "review_count": 89, "in_stock": true, "tags": ["laptop", "ultrabook", "portable"], "release_date": "2023-03-01", "specifications": [{"name": "RAM", "value": "8GB"}, {"name": "Storage", "value": "256GB"}]}
{"index": {}}
{"name": "iPhone 15 Pro", "description": "Latest iPhone with advanced features", "category": "Electronics", "brand": "Apple", "price": 999.99, "rating": 4.7, "review_count": 203, "in_stock": false, "tags": ["smartphone", "premium", "5G"], "release_date": "2023-09-15", "specifications": [{"name": "Storage", "value": "128GB"}, {"name": "Camera", "value": "48MP"}]}

```

### Step 1: Basic Text Search

**Goal:** Find products by name or description

```json
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "laptop",
      "fields": ["name^2", "description"]
    }
  }
}

```

**Key Learning:** Field boosting (`^2`) increases relevance of matches in product names.

### Step 2: Add Filters and Sorting

**Goal:** Find laptops under £1500, sort by rating

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": {
        "multi_match": {
          "query": "laptop",
          "fields": ["name^2", "description"]
        }
      },
      "filter": [
        {"range": {"price": {"lte": 1500}}},
        {"term": {"in_stock": true}}
      ]
    }
  },
  "sort": [
    {"rating": {"order": "desc"}},
    {"review_count": {"order": "desc"}}
  ]
}

```

**Key Learning:** Using `filter` context instead of `must` for non-scoring criteria improves performance.

### Step 3: Add Faceted Search

**Goal:** Provide category and brand facets for filtering

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": {
        "multi_match": {
          "query": "laptop",
          "fields": ["name^2", "description"]
        }
      },
      "filter": [
        {"range": {"price": {"lte": 1500}}}
      ]
    }
  },
  "aggs": {
    "categories": {
      "terms": {
        "field": "category",
        "size": 10
      }
    },
    "brands": {
      "terms": {
        "field": "brand",
        "size": 10
      }
    },
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          {"to": 500},
          {"from": 500, "to": 1000},
          {"from": 1000, "to": 2000},
          {"from": 2000}
        ]
      }
    }
  }
}

```

**Key Learning:** Aggregations provide facets without affecting search results.

### Step 4: Advanced Filtering with Nested Objects

**Goal:** Filter by product specifications

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": {
        "multi_match": {
          "query": "laptop",
          "fields": ["name^2", "description"]
        }
      },
      "filter": [
        {
          "nested": {
            "path": "specifications",
            "query": {
              "bool": {
                "must": [
                  {"term": {"specifications.name": "RAM"}},
                  {"term": {"specifications.value": "16GB"}}
                ]
              }
            }
          }
        }
      ]
    }
  }
}

```

**Key Learning:** Nested queries allow complex filtering on object arrays.

### Step 5: Boost Recent Products

**Goal:** Favour recently released products in search results

```json
GET /products/_search
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": {
            "multi_match": {
              "query": "laptop",
              "fields": ["name^2", "description"]
            }
          },
          "filter": [
            {"range": {"price": {"lte": 1500}}}
          ]
        }
      },
      "functions": [
        {
          "exp": {
            "release_date": {
              "origin": "now",
              "scale": "30d",
              "decay": 0.5
            }
          },
          "weight": 1.5
        },
        {
          "field_value_factor": {
            "field": "rating",
            "factor": 0.2,
            "modifier": "sqrt"
          }
        }
      ],
      "boost_mode": "multiply"
    }
  }
}

```

**Key Learning:** Function score queries combine textual relevance with business logic.

### Step 6: Autocomplete and Search Suggestions

**Goal:** Implement type-ahead search functionality

```json
GET /products/_search
{
  "suggest": {
    "product_suggest": {
      "prefix": "mac",
      "completion": {
        "field": "name.suggest",
        "size": 5,
        "skip_duplicates": true
      }
    }
  }
}

```

**Challenge Extensions:**

1. Add spell correction using fuzzy matching
2. Implement "did you mean" functionality
3. Create product recommendation based on viewing history
4. Optimise for mobile search (shorter queries, location-based)

---

## 📈 Exercise 2: Log Analysis and Monitoring

### Scenario: Application Performance Monitoring

You're building monitoring dashboards for a web application. Progress from simple log queries to complex performance analytics and alerting.

### Sample Data Setup

```json
PUT /app-logs
{
  "mappings": {
    "properties": {
      "@timestamp": {"type": "date"},
      "level": {"type": "keyword"},
      "service": {"type": "keyword"},
      "instance": {"type": "keyword"},
      "user_id": {"type": "keyword"},
      "session_id": {"type": "keyword"},
      "request_id": {"type": "keyword"},
      "endpoint": {"type": "keyword"},
      "method": {"type": "keyword"},
      "status_code": {"type": "integer"},
      "response_time": {"type": "float"},
      "bytes_sent": {"type": "long"},
      "user_agent": {"type": "text"},
      "ip_address": {"type": "ip"},
      "message": {"type": "text"},
      "error_type": {"type": "keyword"},
      "stack_trace": {"type": "text"},
      "tags": {"type": "keyword"}
    }
  }
}

```

### Step 1: Basic Error Detection

**Goal:** Find all error logs in the last hour

```json
GET /app-logs/_search
{
  "query": {
    "bool": {
      "must": [
        {"term": {"level": "ERROR"}},
        {
          "range": {
            "@timestamp": {
              "gte": "now-1h"
            }
          }
        }
      ]
    }
  },
  "sort": [{"@timestamp": {"order": "desc"}}]
}

```

### Step 2: Performance Analysis

**Goal:** Find slow endpoints and calculate percentiles

```json
GET /app-logs/_search
{
  "size": 0,
  "query": {
    "bool": {
      "must": [
        {"term": {"level": "INFO"}},
        {"exists": {"field": "response_time"}},
        {
          "range": {
            "@timestamp": {
              "gte": "now-1h"
            }
          }
        }
      ]
    }
  },
  "aggs": {
    "by_endpoint": {
      "terms": {
        "field": "endpoint",
        "size": 10,
        "order": {"avg_response_time": "desc"}
      },
      "aggs": {
        "avg_response_time": {
          "avg": {"field": "response_time"}
        },
        "response_time_percentiles": {
          "percentiles": {
            "field": "response_time",
            "percents": [50, 90, 95, 99]
          }
        },
        "request_count": {
          "value_count": {"field": "response_time"}
        }
      }
    }
  }
}

```

### Step 3: Error Rate Analysis

**Goal:** Calculate error rates by service and time

```json
GET /app-logs/_search
{
  "size": 0,
  "query": {
    "range": {
      "@timestamp": {
        "gte": "now-24h"
      }
    }
  },
  "aggs": {
    "by_time": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "1h"
      },
      "aggs": {
        "by_service": {
          "terms": {
            "field": "service"
          },
          "aggs": {
            "total_requests": {
              "value_count": {"field": "request_id"}
            },
            "error_requests": {
              "filter": {
                "range": {
                  "status_code": {
                    "gte": 400
                  }
                }
              }
            },
            "error_rate": {
              "bucket_script": {
                "buckets_path": {
                  "errors": "error_requests>_count",
                  "total": "total_requests"
                },
                "script": "params.errors / params.total * 100"
              }
            }
          }
        }
      }
    }
  }
}

```

### Step 4: User Journey Analysis

**Goal:** Track user sessions and identify problem patterns

```json
GET /app-logs/_search
{
  "size": 0,
  "query": {
    "bool": {
      "must": [
        {"exists": {"field": "session_id"}},
        {
          "range": {
            "@timestamp": {
              "gte": "now-1h"
            }
          }
        }
      ]
    }
  },
  "aggs": {
    "by_session": {
      "terms": {
        "field": "session_id",
        "size": 100
      },
      "aggs": {
        "session_duration": {
          "stats": {"field": "@timestamp"}
        },
        "page_views": {
          "value_count": {"field": "endpoint"}
        },
        "error_count": {
          "filter": {
            "term": {"level": "ERROR"}
          }
        },
        "unique_endpoints": {
          "cardinality": {"field": "endpoint"}
        }
      }
    }
  }
}

```

### Step 5: Anomaly Detection

**Goal:** Identify unusual patterns in application behaviour

```json
GET /app-logs/_search
{
  "size": 0,
  "query": {
    "range": {
      "@timestamp": {
        "gte": "now-7d"
      }
    }
  },
  "aggs": {
    "daily_patterns": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "1h"
      },
      "aggs": {
        "request_rate": {
          "value_count": {"field": "request_id"}
        },
        "avg_response_time": {
          "avg": {"field": "response_time"}
        },
        "error_rate": {
          "bucket_script": {
            "buckets_path": {
              "errors": "errors>_count",
              "total": "request_rate"
            },
            "script": "params.total > 0 ? params.errors / params.total * 100 : 0"
          }
        },
        "errors": {
          "filter": {"term": {"level": "ERROR"}}
        }
      }
    }
  }
}

```

**Challenge Extensions:**

1. Implement real-time alerting based on error thresholds
2. Create performance baselines and detect deviations
3. Build user behaviour analytics
4. Implement distributed tracing correlation

---

## 🏥 Exercise 3: Healthcare Data Analytics

### Scenario: Patient Data Analysis and Reporting

You're building analytics for a healthcare system. Progress from basic patient queries to complex population health analytics while maintaining data privacy.

### Sample Data Setup

```json
PUT /patients
{
  "mappings": {
    "properties": {
      "patient_id": {"type": "keyword"},
      "age": {"type": "integer"},
      "gender": {"type": "keyword"},
      "admission_date": {"type": "date"},
      "discharge_date": {"type": "date"},
      "department": {"type": "keyword"},
      "diagnosis_codes": {"type": "keyword"},
      "procedures": {
        "type": "nested",
        "properties": {
          "code": {"type": "keyword"},
          "description": {"type": "text"},
          "date": {"type": "date"},
          "duration_minutes": {"type": "integer"}
        }
      },
      "medications": {
        "type": "nested",
        "properties": {
          "name": {"type": "keyword"},
          "dosage": {"type": "keyword"},
          "start_date": {"type": "date"},
          "end_date": {"type": "date"}
        }
      },
      "length_of_stay": {"type": "integer"},
      "readmission": {"type": "boolean"},
      "discharge_disposition": {"type": "keyword"}
    }
  }
}

```

### Step 1: Basic Patient Cohort Analysis

**Goal:** Find patients by age group and diagnosis

```json
GET /patients/_search
{
  "size": 0,
  "aggs": {
    "age_groups": {
      "range": {
        "field": "age",
        "ranges": [
          {"key": "0-18", "to": 18},
          {"key": "18-65", "from": 18, "to": 65},
          {"key": "65+", "from": 65}
        ]
      },
      "aggs": {
        "common_diagnoses": {
          "terms": {
            "field": "diagnosis_codes",
            "size": 5
          }
        }
      }
    }
  }
}

```

### Step 2: Length of Stay Analysis

**Goal:** Analyze factors affecting length of stay

```json
GET /patients/_search
{
  "size": 0,
  "aggs": {
    "by_department": {
      "terms": {
        "field": "department"
      },
      "aggs": {
        "avg_length_of_stay": {
          "avg": {"field": "length_of_stay"}
        },
        "los_percentiles": {
          "percentiles": {
            "field": "length_of_stay",
            "percents": [25, 50, 75, 90]
          }
        },
        "by_age_group": {
          "range": {
            "field": "age",
            "ranges": [
              {"key": "young", "to": 40},
              {"key": "middle", "from": 40, "to": 65},
              {"key": "elderly", "from": 65}
            ]
          },
          "aggs": {
            "avg_los": {
              "avg": {"field": "length_of_stay"}
            }
          }
        }
      }
    }
  }
}

```

### Step 3: Readmission Risk Analysis

**Goal:** Identify patterns in patient readmissions

```json
GET /patients/_search
{
  "size": 0,
  "query": {
    "range": {
      "discharge_date": {
        "gte": "now-1y"
      }
    }
  },
  "aggs": {
    "readmission_analysis": {
      "filters": {
        "filters": {
          "readmitted": {"term": {"readmission": true}},
          "not_readmitted": {"term": {"readmission": false}}
        }
      },
      "aggs": {
        "by_diagnosis": {
          "terms": {
            "field": "diagnosis_codes",
            "size": 10
          },
          "aggs": {
            "readmission_rate": {
              "filter": {"term": {"readmission": true}}
            }
          }
        },
        "avg_initial_los": {
          "avg": {"field": "length_of_stay"}
        }
      }
    },
    "risk_factors": {
      "terms": {
        "field": "discharge_disposition"
      },
      "aggs": {
        "readmission_rate": {
          "bucket_script": {
            "buckets_path": {
              "readmitted": "readmissions>_count",
              "total": "_count"
            },
            "script": "params.readmitted / params.total * 100"
          }
        },
        "readmissions": {
          "filter": {"term": {"readmission": true}}
        }
      }
    }
  }
}

```

### Step 4: Medication Effectiveness Analysis

**Goal:** Analyze medication patterns and outcomes using nested queries

```json
GET /patients/_search
{
  "size": 0,
  "aggs": {
    "medication_analysis": {
      "nested": {
        "path": "medications"
      },
      "aggs": {
        "by_medication": {
          "terms": {
            "field": "medications.name",
            "size": 20
          },
          "aggs": {
            "patient_outcomes": {
              "reverse_nested": {},
              "aggs": {
                "avg_los": {
                  "avg": {"field": "length_of_stay"}
                },
                "readmission_rate": {
                  "bucket_script": {
                    "buckets_path": {
                      "readmitted": "readmissions>_count",
                      "total": "_count"
                    },
                    "script": "params.total > 0 ? params.readmitted / params.total * 100 : 0"
                  }
                },
                "readmissions": {
                  "filter": {"term": {"readmission": true}}
                }
              }
            }
          }
        }
      }
    }
  }
}

```

### Step 5: Seasonal Health Trends

**Goal:** Identify seasonal patterns in admissions and diagnoses

```json
GET /patients/_search
{
  "size": 0,
  "query": {
    "range": {
      "admission_date": {
        "gte": "now-2y"
      }
    }
  },
  "aggs": {
    "seasonal_trends": {
      "date_histogram": {
        "field": "admission_date",
        "calendar_interval": "month"
      },
      "aggs": {
        "admission_count": {
          "value_count": {"field": "patient_id"}
        },
        "top_diagnoses": {
          "terms": {
            "field": "diagnosis_codes",
            "size": 3
          }
        },
        "emergency_vs_planned": {
          "terms": {
            "field": "department"
          }
        },
        "avg_patient_age": {
          "avg": {"field": "age"}
        }
      }
    }
  }
}

```

**Challenge Extensions:**

1. Implement privacy-preserving analytics (differential privacy)
2. Create predictive models for readmission risk
3. Analyze treatment pathway effectiveness
4. Build population health dashboards

---

## 💡 Progressive Learning Tips

### Building Query Complexity

1. **Start with single conditions** - Master basic match and term queries
2. **Add Boolean logic** - Combine with must, should, must_not
3. **Introduce aggregations** - Start with simple metrics, progress to buckets
4. **Optimize performance** - Use filters, appropriate data types, caching
5. **Add business logic** - Function scores, custom scoring, ML features

### Common Patterns to Master

| Pattern | Use Case | Key Technique |
| --- | --- | --- |
| **Faceted Search** | E-commerce, content sites | Terms aggregations with post-filters |
| **Time Series Analysis** | Logs, metrics, IoT | Date histograms with moving averages |
| **Nested Analytics** | Complex objects, relationships | Nested queries and reverse nested |
| **Anomaly Detection** | Monitoring, fraud detection | Statistical aggregations, ML jobs |
| **Geospatial Analysis** | Location services, logistics | Geo queries and aggregations |

### Performance Optimization Checklist

✅ **Use filter context for non-scoring criteria**

✅ **Limit aggregation cardinality with reasonable size limits**

✅ **Use appropriate data types (keyword vs text)**

✅ **Implement caching for repeated queries**

✅ **Consider index patterns and time-based indices**

✅ **Monitor query performance and slow queries**

---