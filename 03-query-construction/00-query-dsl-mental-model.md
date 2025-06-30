# Query DSL Mental Model

## Building Intuitive Understanding of Query Construction

The Query DSL isn't just syntax to memorise - it's a logical system for expressing what you want to find. Once you understand the patterns, you can construct any query by reasoning through the problem.

---

## 🧩 The Big Picture: Query DSL as a Language

### Understanding the JSON Structure

Every Elasticsearch query follows this logical hierarchy:

```json
{
  "query": {          // WHAT to find
    // Your search logic here
  },
  "aggs": {           // ANALYTICS on the results
    // Your aggregations here
  },
  "sort": [           // HOW to order results
    // Your sorting logic here
  ],
  "size": 10,         // HOW MANY results
  "_source": ["field1", "field2"]  // WHICH FIELDS to return
}

```

**Think of it like asking a librarian:**

- **Query:** "Find me books about cooking"
- **Aggs:** "Also tell me how many are from each decade"
- **Sort:** "Order them by publication date"
- **Size:** "Just show me the first 10"
- **Source:** "I only need title and author"

---

## 🎯 Mental Model #1: Query vs Filter Context

This is the most important concept in Query DSL. There are two fundamentally different ways to ask questions:

### Query Context: "How well does this match?"

```json
{
  "query": {
    "match": {
      "title": "elasticsearch tutorial"
    }
  }
}

```

**Question:** "How relevant is each document to 'elasticsearch tutorial'?"

**Answer:** Documents with scores like 8.5, 6.2, 4.1...

### Filter Context: "Does this match yes/no?"

```json
{
  "query": {
    "bool": {
      "filter": [
        {"term": {"status": "published"}},
        {"range": {"date": {"gte": "2024-01-01"}}}
      ]
    }
  }
}

```

**Question:** "Is the status 'published' AND is the date >= 2024-01-01?"

**Answer:** Yes or No (no scoring)

### When to Use Which

**Use Query Context When:**

- Users are searching with text: "best pizza restaurants"
- Relevance matters: you want the most relevant results first
- Full-text search: content, descriptions, reviews

**Use Filter Context When:**

- Exact matches: status = "active", category = "electronics"
- Ranges: price between $10-50, date in last 30 days
- Boolean conditions: published = true, in_stock = true

**Think First Exercise:**

> A user searches for "red shoes" and you want to show only items under $100 that are in stock. Which parts need query context vs filter context?
> 

**Query context:** "red shoes" (relevance matters - some are more "red shoe-like")

**Filter context:** price < $100 AND in_stock = true (exact conditions, no scoring needed)

```json
{
  "query": {
    "bool": {
      "must": [
        {"match": {"name": "red shoes"}}  // Query context
      ],
      "filter": [
        {"range": {"price": {"lt": 100}}},  // Filter context
        {"term": {"in_stock": true}}        // Filter context
      ]
    }
  }
}

```

---

## 🧠 Mental Model #2: Bool Query as Logical Reasoning

The bool query is your Swiss Army knife. It combines multiple conditions using logic:

### The Four Clauses

```json
{
  "bool": {
    "must": [       // AND - must be true, affects score
      {"match": {"title": "elasticsearch"}}
    ],
    "should": [     // OR - optional, boosts score if present
      {"match": {"tags": "tutorial"}},
      {"match": {"tags": "beginner"}}
    ],
    "must_not": [   // NOT - must not be true, no scoring
      {"term": {"status": "draft"}}
    ],
    "filter": [     // AND - must be true, no scoring
      {"range": {"date": {"gte": "2024-01-01"}}}
    ]
  }
}

```

### Think of it as Logical Statements

**In plain English:**
"Find documents that MUST contain 'elasticsearch' in the title AND MUST be dated after 2024-01-01 AND MUST NOT have status 'draft', and BOOST documents that have 'tutorial' or 'beginner' tags"

**As a logic expression:**`(title contains "elasticsearch") AND (date >= "2024-01-01") AND NOT (status = "draft") AND OPTIONALLY (tags = "tutorial" OR tags = "beginner")`

### Common Patterns

**Pattern 1: Required + Optional**

```json
// Required: must be about cooking
// Optional: boost if it's vegetarian or quick
{
  "bool": {
    "must": [{"match": {"content": "cooking"}}],
    "should": [
      {"match": {"tags": "vegetarian"}},
      {"match": {"tags": "quick"}}
    ]
  }
}

```

**Pattern 2: Required + Filters**

```json
// Required: search term relevance
// Filters: exact conditions that don't affect scoring
{
  "bool": {
    "must": [{"match": {"description": "laptop"}}],
    "filter": [
      {"term": {"category": "electronics"}},
      {"range": {"price": {"gte": 500, "lte": 1500}}}
    ]
  }
}

```

**Pattern 3: Complex Exclusions**

```json
// Find active products except discontinued ones or out of stock
{
  "bool": {
    "must": [{"term": {"status": "active"}}],
    "must_not": [
      {"term": {"discontinued": true}},
      {"term": {"stock_level": 0}}
    ]
  }
}

```

---

## 🔍 Mental Model #3: Query Building Patterns

### Start Simple, Add Complexity

**Step 1: What's the core requirement?**
"Find blog posts about machine learning"

```json
{"match": {"content": "machine learning"}}

```

**Step 2: Add exact conditions**
"...published in 2024"

```json
{
  "bool": {
    "must": [{"match": {"content": "machine learning"}}],
    "filter": [{"range": {"date": {"gte": "2024-01-01"}}}]
  }
}

```

**Step 3: Add exclusions**
"...but not drafts"

```json
{
  "bool": {
    "must": [{"match": {"content": "machine learning"}}],
    "filter": [{"range": {"date": {"gte": "2024-01-01"}}}],
    "must_not": [{"term": {"status": "draft"}}]
  }
}

```

**Step 4: Add optional boosts**
"...prefer posts tagged as 'tutorial'"

```json
{
  "bool": {
    "must": [{"match": {"content": "machine learning"}}],
    "filter": [{"range": {"date": {"gte": "2024-01-01"}}}],
    "must_not": [{"term": {"status": "draft"}}],
    "should": [{"match": {"tags": "tutorial"}}]
  }
}

```

### Common Query Types by Use Case

**Exact Matching (IDs, categories, status)**

```json
{"term": {"status": "published"}}
{"terms": {"category": ["electronics", "computers"]}}

```

**Range Queries (dates, numbers)**

```json
{"range": {"price": {"gte": 10, "lte": 100}}}
{"range": {"date": {"gte": "now-7d"}}}

```

**Text Search (user input, content)**

```json
{"match": {"title": "user input here"}}
{"multi_match": {"query": "search terms", "fields": ["title", "content"]}}

```

**Existence Checks**

```json
{"exists": {"field": "email"}}

```

**Wildcard/Prefix**

```json
{"wildcard": {"username": "john*"}}
{"prefix": {"product_code": "ELEC"}}

```

---

## 🎨 Mental Model #4: Query Composition Patterns

### Pattern 1: The User Search Box

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "{{user_input}}",
            "fields": ["title^3", "description^2", "content"]
          }
        }
      ],
      "filter": [
        {"terms": {"category": ["{{selected_categories}}"]}},
        {"range": {"price": {"gte": "{{min_price}}", "lte": "{{max_price}}"}}}
      ]
    }
  }
}

```

### Pattern 2: The Dashboard Filter

```json
{
  "query": {
    "bool": {
      "filter": [
        {"range": {"@timestamp": {"gte": "{{start_date}}", "lte": "{{end_date}}"}}},
        {"terms": {"service.name": ["{{selected_services}}"]}},
        {"range": {"response_time": {"gte": "{{min_response_time}}"}}}
      ]
    }
  }
}

```

### Pattern 3: The Recommendation Engine

```json
{
  "query": {
    "bool": {
      "should": [
        {"match": {"tags": "{{user_interests}}"}},
        {"terms": {"category": ["{{user_purchase_history}}"]}},
        {"range": {"popularity_score": {"gte": 7}}}
      ],
      "must_not": [
        {"terms": {"id": ["{{already_purchased}}"]}}
      ],
      "minimum_should_match": 1
    }
  }
}

```

---

## 🚀 Building Query Intuition

### The Thought Process

When you need to write a query, think through:

1. **What is the user actually trying to find?** (core must clause)
2. **What are the absolute requirements?** (filter clauses)
3. **What should be excluded?** (must_not clauses)
4. **What would make results better?** (should clauses)
5. **How should results be ordered?** (sort clause)

### Example: E-commerce Product Search

**User searches for:** "wireless headphones"
**Applied filters:** Category = Electronics, Price = $50-$200, In Stock = true
**User preferences:** Prefers Sony brand

**Thought process:**

1. **Core need:** Products matching "wireless headphones" (query context for relevance)
2. **Requirements:** Electronics category, price range, in stock (filter context)
3. **Exclusions:** Nothing specific (no must_not needed)
4. **Preferences:** Sony brand (should clause for boost)

**Resulting query:**

```json
{
  "query": {
    "bool": {
      "must": [
        {"multi_match": {"query": "wireless headphones", "fields": ["name", "description"]}}
      ],
      "filter": [
        {"term": {"category": "electronics"}},
        {"range": {"price": {"gte": 50, "lte": 200}}},
        {"term": {"in_stock": true}}
      ],
      "should": [
        {"term": {"brand": "sony"}}
      ]
    }
  },
  "sort": [
    {"_score": {"order": "desc"}},
    {"popularity": {"order": "desc"}}
  ]
}

```

---

## 🎯 Common Anti-Patterns to Avoid

### ❌ Using query context for exact matches

```json
// Wrong - should not score
{"match": {"status": "published"}}

// Right - exact filter
{"term": {"status": "published"}}

```

### ❌ Using filter context for text search

```json
// Wrong - no relevance scoring
{"term": {"title": "elasticsearch tutorial"}}

// Right - relevance matters
{"match": {"title": "elasticsearch tutorial"}}

```

### ❌ Overly complex single queries

```json
// Hard to read and maintain
{
  "bool": {
    "must": [
      {
        "bool": {
          "should": [
            {"bool": {"must": [/* complex nested logic */]}}
          ]
        }
      }
    ]
  }
}

// Better - break into multiple simpler queries or use aggregations

```

---

## 💡 Practice Exercises

### Exercise 1: Blog Search

**Requirements:** Find blog posts that contain "kubernetes" AND are tagged with either "tutorial" or "guide" AND were published in the last 6 months AND are not drafts.

```json
{
  "query": {
    "bool": {
      "must": [
        {"match": {"content": "kubernetes"}}
      ],
      "filter": [
        {"terms": {"tags": ["tutorial", "guide"]}},
        {"range": {"published_date": {"gte": "now-6M"}}}
      ],
      "must_not": [
        {"term": {"status": "draft"}}
      ]
    }
  }
}

```

### Exercise 2: Product Recommendation

**Requirements:** Find products similar to what user bought before, boost if highly rated, exclude out-of-stock items, prefer products under $50.

```json
{
  "query": {
    "bool": {
      "should": [
        {"terms": {"category": ["user_previous_categories"]}},
        {"range": {"rating": {"gte": 4}}},
        {"range": {"price": {"lte": 50}}}
      ],
      "filter": [
        {"term": {"in_stock": true}}
      ],
      "minimum_should_match": 1
    }
  }
}

```

---

## 🎯 Key Takeaways

1. **Query DSL is logical** - think through what you're asking, don't memorise syntax
2. **Query vs Filter context** - scoring vs exact matching
3. **Bool query is your foundation** - combine conditions logically
4. **Start simple, add complexity** - build queries incrementally
5. **Common patterns exist** - recognise and reuse them

---

## 🚀 Next Steps

Now that you understand the Query DSL structure, let's explore how different types of searches work for different use cases.

**Coming up in Module 3.2:** [Search Patterns by Use Case →](https://claude.ai/chat/link-to-next-module)

---

## 🔧 Quick Reference

### Query Context (Scoring)

- `match` - text search with analysis
- `multi_match` - search across multiple fields
- `match_phrase` - exact phrase matching
- `query_string` - advanced query syntax

### Filter Context (No Scoring)

- `term` - exact value matching
- `terms` - match any of multiple values
- `range` - numeric/date ranges
- `exists` - field has a value
- `bool` - combine multiple conditions

### Structure Template

```json
{
  "query": {
    "bool": {
      "must": [/* required, scored */],
      "filter": [/* required, not scored */],
      "should": [/* optional, boosts score */],
      "must_not": [/* excluded */]
    }
  }
}

```