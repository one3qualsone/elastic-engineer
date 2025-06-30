# Painless Fundamentals

## Elasticsearch's Built-in Scripting Language for Custom Logic

Painless is Elasticsearch's purpose-built scripting language that enables custom logic for scoring, data transformation, and complex calculations. Unlike general-purpose languages, Painless is designed specifically for search and analytics use cases, providing both safety and performance. Understanding Painless unlocks advanced Elasticsearch capabilities that go far beyond standard queries.

---

## 🎯 Why Painless Exists

### The Scripting Challenge in Search Engines

**Traditional approach problems:**

- **Security risks:** General scripting languages can access system resources
- **Performance issues:** Interpreted languages are slow for computation-heavy tasks
- **Memory safety:** Scripts can cause memory leaks or infinite loops
- **API complexity:** Generic languages don't understand Elasticsearch's data structures

**Painless solutions:**

- **Sandboxed execution:** No access to system resources or network
- **Compile-time optimization:** Scripts compiled to Java bytecode for performance
- **Built-in safety:** Automatic circuit breakers and resource limits
- **Elasticsearch-native:** Direct access to document fields and search context

### When to Use Painless

**Perfect for:**

- Custom relevance scoring based on complex business logic
- Data transformation during indexing or search
- Runtime field calculations
- Complex aggregation logic
- Conditional logic that can't be expressed in queries

**Avoid for:**

- Simple operations that standard queries can handle
- Heavy computational tasks (use ingest pipelines instead)
- Operations that could be pre-computed during indexing
- Logic that changes frequently (queries are easier to modify)

---

## 📝 Painless Language Basics

### Syntax and Structure

**Java-like syntax with search-specific features:**

```java
// Variable declarations
String productName = doc['name'].value;
double price = doc['price'].value;
int quantity = doc['quantity'].value;

// Conditional logic
if (price > 100) {
    return quantity * 0.1;  // 10% boost for expensive items
} else {
    return quantity * 0.05; // 5% boost for cheaper items
}

```

**Key differences from Java:**

- **No class declarations:** Scripts are methods, not full classes
- **Simplified imports:** Common classes available by default
- **Document access:** Built-in `doc`, `params`, and `_source` variables
- **Return flexibility:** Can return different types based on context

### Core Data Types

| Type | Declaration | Example Usage |
| --- | --- | --- |
| **boolean** | `boolean flag = true;` | Conditional logic, feature flags |
| **int** | `int count = 42;` | Counters, simple calculations |
| **long** | `long timestamp = 1640995200000L;` | Timestamps, large numbers |
| **float** | `float score = 3.14f;` | Scores, ratings |
| **double** | `double price = 99.99;` | Prices, precise calculations |
| **String** | `String name = "example";` | Text processing, concatenation |
| **List** | `List<String> tags = new ArrayList<>();` | Collections, arrays |
| **Map** | `Map<String, Object> data = new HashMap<>();` | Key-value structures |

### Variable Scope and Context

**Built-in variables in different contexts:**

| Context | Available Variables | Purpose |
| --- | --- | --- |
| **Search scripts** | `doc`, `params`, `_score` | Access document fields and query parameters |
| **Update scripts** | `ctx`, `params` | Modify document source |
| **Ingest scripts** | `ctx`, `params` | Transform documents during indexing |
| **Aggregation scripts** | `doc`, `params`, `_value` | Custom aggregation logic |

---

## 🔍 Document Access Patterns

### The `doc` Variable

**Accessing field values:**

```java
// Single-value fields
String category = doc['category'].value;
double price = doc['price'].value;
long timestamp = doc['@timestamp'].value.millis;

// Multi-value fields (arrays)
List<String> tags = doc['tags'];
if (!tags.empty) {
    String firstTag = tags[0];
    int tagCount = tags.length;
}

// Checking field existence
if (!doc['optional_field'].empty) {
    String value = doc['optional_field'].value;
}

```

**Field access is optimized for performance:**

- Uses doc values (column store) when available
- Cached in memory for repeated access
- Type-safe access to field values

### The `_source` Variable

**Accessing source document:**

```java
// Access nested objects
Map<String, Object> user = (Map<String, Object>) _source['user'];
String userName = (String) user['name'];

// Modify source in update scripts
ctx._source['last_updated'] = new Date();
ctx._source['view_count'] = ((Integer) ctx._source['view_count']) + 1;

// Add new fields
ctx._source['computed_score'] = calculateScore(ctx._source);

```

**When to use `_source` vs `doc`:**

| Use Case | Recommended Approach | Reason |
| --- | --- | --- |
| **Reading simple fields** | `doc['field'].value` | Faster, type-safe |
| **Reading nested objects** | `_source['nested']['field']` | Doc values don't support nesting |
| **Modifying documents** | `ctx._source['field'] = value` | Only way to update source |
| **Complex data structures** | `_source` | Full access to original structure |

---

## ⚡ Performance-Focused Scripting

### Script Compilation and Caching

**How Painless optimizes performance:**

```java
// ✅ Good: Simple, efficient access
double score = doc['rating'].value * params.boost_factor;

// ❌ Avoid: Complex logic that runs for every document
double complexScore = 0;
for (int i = 0; i < 1000; i++) {
    complexScore += Math.log(doc['views'].value + i);
}

```

**Caching best practices:**

```java
// ✅ Use params for values that change per query
double boost = params.category_boost.get(doc['category'].value, 1.0);

// ❌ Don't hardcode values that might change
double boost = doc['category'].value.equals('electronics') ? 1.5 : 1.0;

```

### Efficient Conditional Logic

**Optimized conditional patterns:**

```java
// ✅ Early returns for performance
if (doc['status'].value != 'active') {
    return 0;  // Skip inactive items immediately
}

if (doc['price'].empty) {
    return _score;  // Use original score if no price
}

// ✅ Use ternary operators for simple conditions
double finalPrice = doc['on_sale'].value ?
    doc['sale_price'].value : doc['regular_price'].value;

// ✅ Cache expensive calculations
double baseScore = Math.log(1 + doc['view_count'].value);
return baseScore * (doc['featured'].value ? 2.0 : 1.0);

```

### Mathematical Operations

**Common mathematical patterns:**

```java
// Logarithmic scaling for large numbers
double logViews = Math.log1p(doc['view_count'].value);

// Sigmoid function for bounded scoring
double sigmoid(double x) {
    return 1.0 / (1.0 + Math.exp(-x));
}

// Distance calculations
double distance = Math.sqrt(
    Math.pow(doc['x'].value - params.target_x, 2) +
    Math.pow(doc['y'].value - params.target_y, 2)
);

// Normalization
double normalized = (doc['value'].value - params.min_value) /
    (params.max_value - params.min_value);

```

---

## 🎯 Common Scripting Contexts

### Custom Scoring Scripts

**Basic custom scoring:**

```java
// E-commerce relevance with business logic
double baseScore = _score;
double price = doc['price'].value;
double rating = doc['rating'].empty ? 3.0 : doc['rating'].value;
double viewCount = doc['view_count'].value;

// Price preference (closer to target price = higher score)
double priceScore = 1.0 / (1.0 + Math.abs(price - params.target_price) / 100.0);

// Popularity boost
double popularityScore = Math.log1p(viewCount) / 10.0;

// Quality boost
double qualityScore = rating / 5.0;

// Combine scores
return baseScore * (1.0 + priceScore + popularityScore + qualityScore);

```

**Time-based scoring:**

```java
// Boost recent content
long nowMillis = System.currentTimeMillis();
long docMillis = doc['publish_date'].value.millis;
long ageMillis = nowMillis - docMillis;
long ageDays = ageMillis / (1000 * 60 * 60 * 24);

// Exponential decay over time
double timeBoost = Math.exp(-ageDays / params.half_life_days);

return _score * (1.0 + timeBoost);

```

### Update Scripts

**Increment counters:**

```java
// Safe increment with null checking
if (ctx._source.view_count == null) {
    ctx._source.view_count = 1;
} else {
    ctx._source.view_count++;
}

// Add timestamp
ctx._source.last_viewed = new Date().getTime();

// Update user activity
if (ctx._source.user_activity == null) {
    ctx._source.user_activity = new HashMap();
}
ctx._source.user_activity.put(params.action, new Date().getTime());

```

**Conditional updates:**

```java
// Only update if conditions are met
if (params.new_price < ctx._source.price) {
    ctx._source.price = params.new_price;
    ctx._source.price_updated = new Date();
    ctx._source.discount_applied = true;
} else {
    ctx.op = 'none';  // Don't update document
}

```

### Ingest Pipeline Scripts

**Data enrichment during indexing:**

```java
// Parse user agent string
String userAgent = ctx.user_agent;
if (userAgent.contains('Mobile')) {
    ctx.device_type = 'mobile';
} else if (userAgent.contains('Tablet')) {
    ctx.device_type = 'tablet';
} else {
    ctx.device_type = 'desktop';
}

// Calculate derived fields
ctx.order_total = ctx.quantity * ctx.unit_price;
ctx.order_category = ctx.order_total > 100 ? 'large' : 'small';

// Clean and standardize data
ctx.email = ctx.email.toLowerCase().trim();
ctx.phone = ctx.phone.replaceAll('[^0-9]', '');

```

---

## 🛠️ Advanced Painless Techniques

### Working with Collections

**List operations:**

```java
// Process array fields
List<String> categories = doc['categories'];
List<String> processedCategories = new ArrayList();

for (String category : categories) {
    if (category.length() > 3) {
        processedCategories.add(category.toLowerCase());
    }
}

// Count matching elements
int electronicsCount = 0;
for (String category : categories) {
    if (category.equals('electronics')) {
        electronicsCount++;
    }
}

// Find maximum value in array
List<Double> prices = doc['variant_prices'];
double maxPrice = 0;
for (double price : prices) {
    if (price > maxPrice) {
        maxPrice = price;
    }
}

```

**Map operations:**

```java
// Process nested objects
Map<String, Object> attributes = (Map<String, Object>) _source['attributes'];
double totalScore = 0;

for (String key : attributes.keySet()) {
    Object value = attributes.get(key);
    if (value instanceof Number) {
        totalScore += ((Number) value).doubleValue();
    }
}

// Build lookup maps for efficiency
Map<String, Double> categoryBoosts = params.category_boosts;
String docCategory = doc['category'].value;
double boost = categoryBoosts.getOrDefault(docCategory, 1.0);

```

### String Processing

**Text analysis and manipulation:**

```java
// String analysis
String content = doc['content'].value;
String[] words = content.split('\\s+');
int wordCount = words.length;

// Find keywords
int keywordCount = 0;
List<String> keywords = params.keywords;
for (String word : words) {
    if (keywords.contains(word.toLowerCase())) {
        keywordCount++;
    }
}

// Calculate keyword density
double keywordDensity = (double) keywordCount / wordCount;

// Text cleaning
String cleanedText = content
    .toLowerCase()
    .replaceAll('[^a-z0-9\\s]', '')
    .trim();

```

### Date and Time Operations

**Temporal calculations:**

```java
// Parse dates
long publishTime = doc['publish_date'].value.millis;
long currentTime = System.currentTimeMillis();

// Calculate age in different units
long ageMillis = currentTime - publishTime;
long ageHours = ageMillis / (1000 * 60 * 60);
long ageDays = ageHours / 24;

// Business hours calculation
Calendar cal = Calendar.getInstance();
cal.setTimeInMillis(publishTime);
int hourOfDay = cal.get(Calendar.HOUR_OF_DAY);
boolean isBusinessHours = hourOfDay >= 9 && hourOfDay <= 17;

// Seasonal adjustments
int month = cal.get(Calendar.MONTH);
double seasonalBoost = 1.0;
if (month >= 10 || month <= 1) {  // Winter months
    seasonalBoost = 1.2;
}

```

---

## 🚫 Common Pitfalls and Best Practices

### Performance Anti-patterns

**❌ Expensive operations in loops:**

```java
// Don't do this - expensive regex in loop
List<String> items = doc['items'];
int matchCount = 0;
for (String item : items) {
    if (item.matches(".*expensive.*")) {  // Slow regex
        matchCount++;
    }
}

```

**✅ Optimized version:**

```java
// Pre-compile patterns or use simple string operations
List<String> items = doc['items'];
int matchCount = 0;
for (String item : items) {
    if (item.contains("expensive")) {  // Fast string contains
        matchCount++;
    }
}

```

### Memory Management

**❌ Creating unnecessary objects:**

```java
// Avoid creating objects in loops
for (int i = 0; i < 1000; i++) {
    String temp = "value_" + i;  // Creates many string objects
    // ... processing
}

```

**✅ Reuse objects when possible:**

```java
// Pre-calculate or use StringBuilder for string building
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.setLength(0);  // Reset instead of creating new
    sb.append("value_").append(i);
    String temp = sb.toString();
    // ... processing
}

```

### Error Handling

**Robust field access:**

```java
// ✅ Safe field access with defaults
double rating = doc['rating'].empty ?
    params.default_rating : doc['rating'].value;

// ✅ Check for null values in source
Object priceObj = _source['price'];
double price = (priceObj != null) ?
    ((Number) priceObj).doubleValue() : 0.0;

// ✅ Graceful degradation
try {
    double complexCalculation = performComplexMath(doc['values']);
    return complexCalculation;
} catch (Exception e) {
    return _score;  // Fall back to original score
}

```

### Parameter Usage

**Efficient parameter patterns:**

```java
// ✅ Use params for configuration
Map<String, Double> weights = params.field_weights;
double score = 0;
for (String field : weights.keySet()) {
    if (!doc[field].empty) {
        score += doc[field].value * weights.get(field);
    }
}

// ✅ Cache expensive lookups
Map<String, Double> categoryMultipliers = params.category_multipliers;
String category = doc['category'].value;
double multiplier = categoryMultipliers.getOrDefault(category, 1.0);

```

---

## 🧪 Testing and Debugging Painless Scripts

### Script Testing Strategies

**Test scripts in isolation:**

```json
POST /_scripts/painless/_execute
{
  "script": {
    "source": """
      double price = doc['price'].value;
      double rating = doc['rating'].value;
      return price * rating * params.boost;
    """,
    "params": {
      "boost": 1.5
    }
  },
  "context": "score",
  "context_setup": {
    "index": "products",
    "document": {
      "price": 99.99,
      "rating": 4.5
    }
  }
}

```

**Debug with logging:**

```java
// Add debug output (remove in production)
double score = doc['rating'].value * params.boost;
Debug.explain("Calculated score: " + score);
return score;

```

### Performance Testing

**Benchmark script performance:**

```python
import time
from elasticsearch import Elasticsearch

def benchmark_script_performance():
    es = Elasticsearch()

    # Test query with script
    script_query = {
        "query": {
            "function_score": {
                "query": {"match_all": {}},
                "script_score": {
                    "script": {
                        "source": """
                            Math.log(1 + doc['view_count'].value) * params.boost
                        """,
                        "params": {"boost": 1.5}
                    }
                }
            }
        },
        "size": 100
    }

    # Baseline query without script
    baseline_query = {
        "query": {"match_all": {}},
        "size": 100
    }

    # Run tests
    script_times = []
    baseline_times = []

    for i in range(10):
        # Test with script
        start = time.time()
        es.search(index="products", body=script_query)
        script_times.append((time.time() - start) * 1000)

        # Test baseline
        start = time.time()
        es.search(index="products", body=baseline_query)
        baseline_times.append((time.time() - start) * 1000)

    script_avg = sum(script_times) / len(script_times)
    baseline_avg = sum(baseline_times) / len(baseline_times)
    overhead = script_avg - baseline_avg

    print(f"Script avg: {script_avg:.2f}ms")
    print(f"Baseline avg: {baseline_avg:.2f}ms")
    print(f"Script overhead: {overhead:.2f}ms ({overhead/baseline_avg*100:.1f}%)")

```

---

## 💡 Key Takeaways

✅ **Painless is optimized for search-specific operations—use it for complex scoring and transformations**

✅ **Prefer `doc` values over `_source` for simple field access—it's faster and type-safe**

✅ **Use parameters to make scripts flexible and cache-friendly**

✅ **Keep scripts simple and avoid expensive operations in loops**

✅ **Test script performance impact before deploying to production**

✅ **Handle edge cases gracefully with proper null checking and defaults**

✅ **Consider pre-computing complex values during indexing instead of calculating at search time**

---