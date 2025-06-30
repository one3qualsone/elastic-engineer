# Document Design Principles

## Structuring Documents for Optimal Search Performance

Document design is where search performance is won or lost. The decisions you make about how to structure your documents will determine what's possible and what's fast in your Elasticsearch implementation.

---

## 🎯 The Core Principle: Design for How You Search

### The Fundamental Question

Before modeling any document, ask: **"How will users search for this information?"**

**Wrong approach:** Model documents like database tables

```json
// Database thinking - normalized structure
{
  "user_id": 12345,
  "order_id": 67890,
  "product_id": 54321
}

```

**Right approach:** Model documents for search scenarios

```json
// Search thinking - denormalized for discoverability
{
  "order_id": "67890",
  "customer": {
    "name": "John Smith",
    "email": "john@example.com",
    "tier": "premium",
    "location": "San Francisco, CA"
  },
  "products": [
    {
      "name": "Wireless Headphones",
      "category": "Electronics",
      "brand": "SoundTech",
      "price": 199.99
    }
  ],
  "order_date": "2024-01-15",
  "total_amount": 199.99,
  "status": "delivered"
}

```

**Why the second approach works better:**

- Search for "premium customers in San Francisco" → One query
- Search for "SoundTech orders" → Direct match
- Search for "delivered orders over $150" → All data in one place
- Search for "John Smith's orders" → Complete information available

---

## 📝 Denormalization: Embrace Data Duplication

### Why Denormalization Wins in Search

**The relational mindset:**

- Minimize data duplication
- Normalize into separate tables
- Use joins to combine data

**The search mindset:**

- Duplicate data for search convenience
- Keep related information together
- Optimize for read performance

### Denormalization Decision Framework

| Question | Normalize (Separate) | Denormalize (Include) |
| --- | --- | --- |
| How often does this data change? | Frequently | Rarely |
| Do searches need this data? | No | Yes |
| Is this data large? | Very large | Moderate size |
| Is consistency critical? | Yes | No |

**Example - Product Catalog:**

**Scenario:** E-commerce products with categories and brands

**Option 1 - Normalized (like a database):**

```json
// Products index
{"product_id": "P123", "name": "iPhone", "category_id": "C1", "brand_id": "B1"}

// Categories index
{"category_id": "C1", "name": "Smartphones", "description": "Mobile phones"}

// Brands index
{"brand_id": "B1", "name": "Apple", "country": "USA"}

```

**Problems with Option 1:**

- Need multiple queries to get complete product information
- Can't search "Apple smartphones" in a single query
- Category and brand filters require expensive operations

**Option 2 - Denormalized (search-optimized):**

```json
{
  "product_id": "P123",
  "name": "iPhone 15 Pro",
  "description": "Latest Apple smartphone with advanced features",
  "category": {
    "id": "C1",
    "name": "Smartphones",
    "path": "Electronics > Mobile > Smartphones"
  },
  "brand": {
    "id": "B1",
    "name": "Apple",
    "country": "USA"
  },
  "price": 999.99,
  "features": ["5G", "Face ID", "Triple Camera"],
  "specifications": {
    "screen_size": "6.1 inch",
    "storage": "128GB",
    "color": "Space Black"
  }
}

```

**Benefits of Option 2:**

- Single query gets complete product information
- Search "Apple smartphones under $1000" works perfectly
- Faceted search on brand, category, features works immediately
- Much faster user experience

### When NOT to Denormalize

**Keep separate when:**

- Data changes very frequently (stock levels, real-time prices)
- Data is extremely large (full product manuals, high-res images)
- Strict consistency is required (financial balances, inventory counts)
- The data isn't needed for search (internal system metadata)

**Example - User Account Balance:**

```json
// Good - keep financial data separate
{
  "user_id": "U123",
  "name": "John Smith",
  "account_type": "premium"
  // Don't include: account_balance (changes too frequently)
}

// Query account balance separately when needed
GET users/_doc/U123
GET accounts/_doc/ACC_U123

```

---

## 🔍 Optimizing for Search Patterns

### Pattern 1: Full-Text Search Documents

**Use case:** Blog posts, articles, documentation

**Design principles:**

- Include all searchable text in the document
- Add metadata that users filter by
- Structure for relevance boosting

**Example - Blog Post:**

```json
{
  "post_id": "blog_123",
  "title": "Getting Started with Elasticsearch",
  "content": "Elasticsearch is a powerful search engine...",
  "excerpt": "Learn the basics of Elasticsearch in this comprehensive guide",
  "author": {
    "name": "Jane Smith",
    "bio": "Senior Engineer with 10 years experience",
    "expertise": ["search", "databases", "performance"]
  },
  "metadata": {
    "publish_date": "2024-01-15",
    "last_updated": "2024-01-20",
    "reading_time_minutes": 8,
    "difficulty": "beginner"
  },
  "tags": ["elasticsearch", "search", "tutorial", "beginners"],
  "category": {
    "primary": "Technology",
    "secondary": "Search Engines"
  },
  "engagement": {
    "view_count": 1247,
    "like_count": 89,
    "comment_count": 23
  }
}

```

**Why this structure works:**

- **Title boost:** Most important for relevance
- **Content searchable:** Full-text search across all content
- **Author expertise:** Find posts by experts in specific areas
- **Metadata filtering:** Filter by date, difficulty, reading time
- **Engagement signals:** Boost popular content

### Pattern 2: Faceted Search Documents

**Use case:** Product catalogs, job listings, real estate

**Design principles:**

- Include all filterable attributes
- Structure for aggregations
- Support range and term filtering

**Example - Job Listing:**

```json
{
  "job_id": "job_456",
  "title": "Senior Software Engineer",
  "description": "We're looking for an experienced developer...",
  "company": {
    "name": "TechCorp",
    "size": "500-1000",
    "industry": "Technology",
    "location": {
      "city": "San Francisco",
      "state": "CA",
      "country": "USA",
      "coordinates": {"lat": 37.7749, "lon": -122.4194}
    }
  },
  "requirements": {
    "experience_years": {"min": 5, "max": 10},
    "education": "Bachelor's degree",
    "skills_required": ["Python", "JavaScript", "AWS"],
    "skills_preferred": ["React", "Docker", "Kubernetes"]
  },
  "compensation": {
    "salary_range": {"min": 120000, "max": 180000},
    "currency": "USD",
    "equity": true,
    "benefits": ["health", "dental", "401k", "remote_work"]
  },
  "job_details": {
    "employment_type": "full_time",
    "remote_policy": "hybrid",
    "posted_date": "2024-01-15",
    "application_deadline": "2024-02-15"
  }
}

```

**Search scenarios this enables:**

- "Python jobs in San Francisco" → location + skills filtering
- "Remote senior developer roles $150k+" → remote + experience + salary
- "Jobs at tech companies with equity" → industry + compensation filtering

### Pattern 3: Time-Series Documents

**Use case:** Logs, metrics, events, sensor data

**Design principles:**

- Always include timestamp
- Structure for time-based aggregations
- Include all context needed for analysis

**Example - Application Log:**

```json
{
  "@timestamp": "2024-01-15T10:30:45.123Z",
  "service": {
    "name": "user-service",
    "version": "2.1.4",
    "environment": "production",
    "instance": "us-west-2a-i123"
  },
  "request": {
    "id": "req_abc123",
    "method": "POST",
    "url": "/api/users/create",
    "user_agent": "Mozilla/5.0...",
    "ip_address": "192.168.1.100"
  },
  "response": {
    "status_code": 201,
    "duration_ms": 245,
    "body_size_bytes": 1024
  },
  "user": {
    "id": "user_789",
    "session_id": "session_xyz"
  },
  "message": "User created successfully",
  "level": "INFO",
  "tags": ["user-creation", "api-success"]
}

```

**Why this structure works:**

- **Time-based analysis:** Group by time periods
- **Service filtering:** Analyze specific services or environments
- **Performance monitoring:** Track response times and error rates
- **User tracking:** Follow user journeys across requests
- **Debugging context:** All information needed to investigate issues

---

## 🎨 Field Design Best Practices

### Text Fields: Designed for Discovery

**Key decisions for text fields:**

- What analyzer to use?
- Should it be searchable, aggregatable, or both?
- What boost factor for relevance?

**Example - Product Description:**

```json
{
  "description": {
    "type": "text",
    "analyzer": "english",
    "fields": {
      "keyword": {
        "type": "keyword",
        "ignore_above": 256
      },
      "search": {
        "type": "text",
        "analyzer": "standard"
      }
    }
  }
}

```

**Why multiple fields:**

- **Main field:** English analyzer for semantic search
- **Keyword field:** Exact matching and aggregations
- **Search field:** Standard analyzer for broader matching

### Keyword Fields: Designed for Filtering

**Use keyword fields for:**

- Status values: "active", "pending", "cancelled"
- Categories: "electronics", "clothing", "books"
- IDs: user IDs, product codes, transaction IDs
- Exact matches: email addresses, usernames

**Example - Status and Category Fields:**

```json
{
  "status": {
    "type": "keyword"
  },
  "category": {
    "type": "keyword",
    "fields": {
      "text": {
        "type": "text",
        "analyzer": "standard"
      }
    }
  }
}

```

### Numeric and Date Fields: Designed for Ranges

**Design considerations:**

- What's the appropriate numeric type?
- Do you need both exact values and ranges?
- Should dates include timezone information?

**Example - Price and Date Fields:**

```json
{
  "price": {
    "type": "scaled_float",
    "scaling_factor": 100
  },
  "created_date": {
    "type": "date",
    "format": "yyyy-MM-dd'T'HH:mm:ss.SSSZ"
  },
  "price_ranges": {
    "budget": {"type": "boolean"},
    "mid_range": {"type": "boolean"},
    "premium": {"type": "boolean"}
  }
}

```

**Why scaled_float:** More storage-efficient than double for prices

**Why price ranges:** Enable fast filtering without range calculations

---

## 🔗 Handling Relationships in Documents

### One-to-Many: Nested Objects vs Arrays

**Scenario:** Product with multiple variants (size, color, price)

**Option 1 - Simple Array (loses relationships):**

```json
{
  "product": "T-Shirt",
  "sizes": ["S", "M", "L"],
  "colors": ["red", "blue", "green"],
  "prices": [19.99, 24.99, 29.99]
}

```

**Problem:** Can't tell which price goes with which size/color combination.

**Option 2 - Nested Objects (preserves relationships):**

```json
{
  "product": "T-Shirt",
  "variants": [
    {"size": "S", "color": "red", "price": 19.99, "stock": 10},
    {"size": "M", "color": "blue", "price": 24.99, "stock": 5},
    {"size": "L", "color": "green", "price": 29.99, "stock": 0}
  ]
}

```

**Nested field mapping:**

```json
{
  "variants": {
    "type": "nested",
    "properties": {
      "size": {"type": "keyword"},
      "color": {"type": "keyword"},
      "price": {"type": "scaled_float", "scaling_factor": 100},
      "stock": {"type": "integer"}
    }
  }
}

```

### Many-to-Many: When to Flatten vs Reference

**Scenario:** Articles with multiple authors, authors write multiple articles

**Option 1 - Embed Author Info (good for search):**

```json
{
  "article_id": "A123",
  "title": "Advanced Elasticsearch Techniques",
  "authors": [
    {
      "id": "author_1",
      "name": "Jane Smith",
      "expertise": ["search", "databases"],
      "bio": "Senior Engineer with 10 years experience"
    },
    {
      "id": "author_2",
      "name": "Bob Johnson",
      "expertise": ["performance", "scaling"],
      "bio": "Principal Architect"
    }
  ],
  "content": "..."
}

```

**When to use:** Author info is relatively stable, and you need to search by author expertise or bio.

**Option 2 - Reference Author IDs (good for consistency):**

```json
{
  "article_id": "A123",
  "title": "Advanced Elasticsearch Techniques",
  "author_ids": ["author_1", "author_2"],
  "content": "..."
}

```

**When to use:** Author info changes frequently, or you need strict consistency across articles.

---

## 📊 Document Size and Performance Considerations

### Optimal Document Size

**Document size guidelines:**

| Size Range | Implications | Recommendations |
| --- | --- | --- |
| **< 1KB** | Very fast, minimal overhead | Good for simple records |
| **1-10KB** | Good balance of speed and content | Ideal for most use cases |
| **10-100KB** | Slower indexing, more memory usage | Acceptable for rich documents |
| **> 100KB** | Significant performance impact | Consider splitting or external storage |

### Large Document Strategies

**Strategy 1 - Content Splitting:**

```json
// Main document - searchable metadata
{
  "document_id": "DOC123",
  "title": "Product Manual",
  "summary": "Complete guide for product usage",
  "metadata": {...},
  "sections": [
    {"id": "S1", "title": "Getting Started"},
    {"id": "S2", "title": "Advanced Features"}
  ]
}

// Section documents - detailed content
{
  "document_id": "DOC123",
  "section_id": "S1",
  "title": "Getting Started",
  "content": "Detailed step-by-step instructions..."
}

```

**Strategy 2 - External Content Storage:**

```json
{
  "document_id": "DOC123",
  "title": "Product Manual",
  "summary": "Complete guide for product usage",
  "metadata": {...},
  "content_location": {
    "type": "s3",
    "bucket": "documents",
    "key": "manuals/DOC123.pdf"
  }
}

```

### Performance Impact of Document Structure

**Factors affecting performance:**

| Factor | Impact | Optimization |
| --- | --- | --- |
| **Number of fields** | Memory usage increases | Only include searchable fields |
| **Text field size** | Analysis time increases | Consider length limits |
| **Nested object depth** | Query complexity increases | Keep nesting shallow |
| **Array size** | Memory and processing overhead | Limit array lengths |

---

## 💡 Key Takeaways

✅ **Design documents for how users will search, not how databases store**

✅ **Denormalize aggressively - duplicate data for search convenience**

✅ **Include all searchable and filterable information in each document**

✅ **Choose field types based on how they'll be used (search vs filter vs aggregate)**

✅ **Use nested objects when relationships between array elements matter**

✅ **Keep documents under 10KB when possible for optimal performance**

✅ **Consider splitting very large documents or storing content externally**

---