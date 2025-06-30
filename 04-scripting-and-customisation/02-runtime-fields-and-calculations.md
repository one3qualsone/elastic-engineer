# Runtime Fields and Calculations

## Dynamic Fields Without Reindexing

Runtime fields represent a paradigm shift in how we think about schema evolution and computed fields. Instead of pre-computing values during indexing, runtime fields calculate values on-demand during search. This enables dynamic schema changes, complex calculations, and flexible data exploration without the cost and complexity of reindexing.

---

## 🔄 The Runtime Field Revolution

### Traditional Approach vs Runtime Fields

**Traditional field addition:**

```
1. Identify new field requirement
2. Update mapping in new index
3. Reindex all data (hours/days for large datasets)
4. Update application to use new index
5. Switch aliases and cleanup

```

**Runtime field approach:**

```
1. Define runtime field in mapping or query
2. Field immediately available for search/aggregation
3. No reindexing required
4. No downtime

```

### When Runtime Fields Excel

**Perfect scenarios:**

- **Exploratory analysis:** Test field calculations before committing to reindexing
- **Schema evolution:** Add fields to existing indices without downtime
- **Complex calculations:** Combine multiple fields with business logic
- **Temporary fields:** Short-term analysis needs
- **Cross-field computations:** Calculations that span multiple existing fields

**Consider alternatives when:**

- **High query frequency:** Pre-computed fields are faster for frequent operations
- **Simple field access:** Direct field access is more efficient
- **Storage costs are low:** Pre-computing may be worth the storage cost
- **Complex joins:** Some operations are too expensive to compute repeatedly

---

## 🛠️ Runtime Field Fundamentals

### Basic Runtime Field Definition

**In index mapping:**

```json
{
  "mappings": {
    "runtime": {
      "full_name": {
        "type": "keyword",
        "script": {
          "source": "emit(doc['first_name'].value + ' ' + doc['last_name'].value)"
        }
      },
      "price_category": {
        "type": "keyword",
        "script": {
          "source": """
            double price = doc['price'].value;
            if (price < 50) {
              emit('budget');
            } else if (price < 200) {
              emit('mid-range');
            } else {
              emit('premium');
            }
          """
        }
      }
    },
    "properties": {
      "first_name": {"type": "keyword"},
      "last_name": {"type": "keyword"},
      "price": {"type": "double"}
    }
  }
}

```

**In search query:**

```json
{
  "runtime_mappings": {
    "discount_percentage": {
      "type": "double",
      "script": {
        "source": """
          if (doc['sale_price'].size() > 0 && doc['regular_price'].size() > 0) {
            double regular = doc['regular_price'].value;
            double sale = doc['sale_price'].value;
            emit(((regular - sale) / regular) * 100);
          }
        """
      }
    }
  },
  "query": {
    "range": {
      "discount_percentage": {"gte": 20}
    }
  }
}

```

### Runtime Field Types

| Type | Use Cases | Example |
| --- | --- | --- |
| **keyword** | Categories, IDs, exact matching | Status classification, user segments |
| **long** | Counters, IDs, timestamps | Age calculation, days since event |
| **double** | Calculations, scores, ratios | Price calculations, performance metrics |
| **date** | Date calculations, formatting | Derived dates, time zone conversions |
| **boolean** | Flags, conditions | Feature flags, eligibility checks |
| **ip** | IP calculations | Network analysis, geolocation |
| **geo_point** | Location calculations | Distance calculations, region mapping |

---

## 📊 Business Logic Runtime Fields

### E-commerce Calculations

**Product classification and pricing:**

```java
// Product tier classification
String brand = doc['brand'].value;
double price = doc['price'].value;
String category = doc['category'].value;

String tier = 'standard';

// Luxury brands
if (['gucci', 'prada', 'versace'].contains(brand.toLowerCase())) {
    tier = 'luxury';
}
// Premium pricing
else if (price > 500) {
    tier = 'premium';
}
// Budget category
else if (price < 50) {
    tier = 'budget';
}
// Category-specific logic
else if (category.equals('electronics') && price > 200) {
    tier = 'premium';
}

emit(tier);

```

**Dynamic pricing and promotions:**

```java
// Calculate effective price with promotions
double basePrice = doc['base_price'].value;
double currentPrice = basePrice;

// Apply member discounts
if (doc['customer_tier'].size() > 0) {
    String tier = doc['customer_tier'].value;
    if (tier.equals('gold')) {
        currentPrice *= 0.9;  // 10% discount
    } else if (tier.equals('platinum')) {
        currentPrice *= 0.85; // 15% discount
    }
}

// Apply seasonal promotions
long now = System.currentTimeMillis();
long summerStart = 1622505600000L; // June 1st
long summerEnd = 1630454400000L;   // September 1st

if (now >= summerStart && now <= summerEnd) {
    String category = doc['category'].value;
    if (['swimwear', 'outdoors', 'travel'].contains(category)) {
        currentPrice *= 0.8; // 20% summer discount
    }
}

// Apply clearance pricing for old inventory
if (doc['inventory_age_days'].size() > 0) {
    int ageDays = doc['inventory_age_days'].value;
    if (ageDays > 90) {
        currentPrice *= 0.7; // 30% clearance
    } else if (ageDays > 60) {
        currentPrice *= 0.85; // 15% older inventory
    }
}

emit(Math.round(currentPrice * 100.0) / 100.0); // Round to 2 decimals

```

### Customer Analytics Fields

**Customer lifetime value calculation:**

```java
// Calculate customer lifetime value score
double totalSpent = doc['total_order_value'].value;
int orderCount = doc['order_count'].value;
long firstOrderMillis = doc['first_order_date'].value.millis;
long lastOrderMillis = doc['last_order_date'].value.millis;

// Customer tenure in months
long tenureMonths = (lastOrderMillis - firstOrderMillis) / (1000L * 60 * 60 * 24 * 30);
if (tenureMonths == 0) tenureMonths = 1; // Minimum 1 month

// Average order value
double avgOrderValue = totalSpent / orderCount;

// Purchase frequency (orders per month)
double orderFrequency = (double) orderCount / tenureMonths;

// Recency factor (recent activity = higher value)
long daysSinceLastOrder = (System.currentTimeMillis() - lastOrderMillis) / (1000L * 60 * 60 * 24);
double recencyFactor = Math.max(0.1, 1.0 - (daysSinceLastOrder / 365.0)); // Decay over 1 year

// CLV calculation
double clv = avgOrderValue * orderFrequency * Math.sqrt(tenureMonths) * recencyFactor;

emit(Math.round(clv * 100.0) / 100.0);

```

**Customer segmentation:**

```java
// Multi-dimensional customer segmentation
double totalSpent = doc['total_order_value'].value;
int orderCount = doc['order_count'].value;
long daysSinceLastOrder = (System.currentTimeMillis() - doc['last_order_date'].value.millis) / (1000L * 60 * 60 * 24);

String segment = 'inactive';

if (daysSinceLastOrder <= 30) {
    if (totalSpent >= 1000 && orderCount >= 10) {
        segment = 'champion';
    } else if (totalSpent >= 500 || orderCount >= 5) {
        segment = 'loyal';
    } else {
        segment = 'new_customer';
    }
} else if (daysSinceLastOrder <= 90) {
    if (totalSpent >= 1000) {
        segment = 'potential_loyalist';
    } else {
        segment = 'regular';
    }
} else if (daysSinceLastOrder <= 180) {
    if (totalSpent >= 500) {
        segment = 'at_risk';
    } else {
        segment = 'hibernating';
    }
} else {
    if (totalSpent >= 1000) {
        segment = 'cant_lose_them';
    } else {
        segment = 'lost';
    }
}

emit(segment);

```

---

## 📅 Time-Based Runtime Fields

### Date Calculations and Transformations

**Age and tenure calculations:**

```java
// Calculate age from birth date
if (doc['birth_date'].size() > 0) {
    long birthMillis = doc['birth_date'].value.millis;
    long nowMillis = System.currentTimeMillis();
    long ageMillis = nowMillis - birthMillis;
    long ageYears = ageMillis / (1000L * 60 * 60 * 24 * 365);
    emit(ageYears);
} else {
    emit(null);
}

```

**Business time calculations:**

```java
// Calculate business days between dates
if (doc['start_date'].size() > 0 && doc['end_date'].size() > 0) {
    long startMillis = doc['start_date'].value.millis;
    long endMillis = doc['end_date'].value.millis;

    Calendar start = Calendar.getInstance();
    Calendar end = Calendar.getInstance();
    start.setTimeInMillis(startMillis);
    end.setTimeInMillis(endMillis);

    int businessDays = 0;
    Calendar current = (Calendar) start.clone();

    while (current.before(end) || current.equals(end)) {
        int dayOfWeek = current.get(Calendar.DAY_OF_WEEK);
        // Monday = 2, Friday = 6
        if (dayOfWeek >= 2 && dayOfWeek <= 6) {
            businessDays++;
        }
        current.add(Calendar.DAY_OF_MONTH, 1);
    }

    emit(businessDays);
}

```

**Seasonal and periodic calculations:**

```java
// Determine season and quarter
if (doc['date'].size() > 0) {
    Calendar cal = Calendar.getInstance();
    cal.setTimeInMillis(doc['date'].value.millis);

    int month = cal.get(Calendar.MONTH); // 0-based

    String season;
    String quarter;

    if (month >= 2 && month <= 4) {        // Mar-May
        season = 'spring';
        quarter = 'Q2';
    } else if (month >= 5 && month <= 7) { // Jun-Aug
        season = 'summer';
        quarter = 'Q3';
    } else if (month >= 8 && month <= 10) { // Sep-Nov
        season = 'autumn';
        quarter = 'Q4';
    } else {                               // Dec-Feb
        season = 'winter';
        quarter = month == 11 ? 'Q4' : 'Q1';
    }

    emit(season + '_' + quarter);
}

```

### Time Zone Conversions

**Multi-timezone support:**

```java
// Convert UTC timestamp to user timezone
if (doc['timestamp_utc'].size() > 0 && doc['user_timezone'].size() > 0) {
    long utcMillis = doc['timestamp_utc'].value.millis;
    String userTz = doc['user_timezone'].value;

    // Simple timezone offset lookup (in production, use proper timezone library)
    Map<String, Integer> tzOffsets = [
        'America/New_York': -5,
        'America/Los_Angeles': -8,
        'Europe/London': 0,
        'Europe/Berlin': 1,
        'Asia/Tokyo': 9,
        'Australia/Sydney': 10
    ];

    Integer offsetHours = tzOffsets.get(userTz);
    if (offsetHours != null) {
        long localMillis = utcMillis + (offsetHours * 60 * 60 * 1000);
        emit(localMillis);
    } else {
        emit(utcMillis); // Fallback to UTC
    }
}

```

---

## 🌍 Geographic Runtime Fields

### Distance and Location Calculations

**Distance from user location:**

```java
// Calculate distance from user's location
if (doc['location'].size() > 0 && params.user_lat != null && params.user_lon != null) {
    double venueLat = doc['location'].lat;
    double venueLon = doc['location'].lon;
    double userLat = params.user_lat;
    double userLon = params.user_lon;

    // Haversine formula for great-circle distance
    double R = 6371; // Earth's radius in kilometers
    double dLat = Math.toRadians(venueLat - userLat);
    double dLon = Math.toRadians(venueLon - userLon);

    double a = Math.sin(dLat/2) * Math.sin(dLat/2) +
               Math.cos(Math.toRadians(userLat)) * Math.cos(Math.toRadians(venueLat)) *
               Math.sin(dLon/2) * Math.sin(dLon/2);
    double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
    double distance = R * c;

    emit(Math.round(distance * 100.0) / 100.0); // Round to 2 decimals
}

```

**Geographic region classification:**

```java
// Classify location into geographic regions
if (doc['location'].size() > 0) {
    double lat = doc['location'].lat;
    double lon = doc['location'].lon;

    String region = 'unknown';

    // North America
    if (lat >= 25 && lat <= 71 && lon >= -168 && lon <= -52) {
        region = 'north_america';
    }
    // Europe
    else if (lat >= 35 && lat <= 71 && lon >= -25 && lon <= 45) {
        region = 'europe';
    }
    // Asia
    else if (lat >= -10 && lat <= 71 && lon >= 45 && lon <= 180) {
        region = 'asia';
    }
    // Australia/Oceania
    else if (lat >= -47 && lat <= -10 && lon >= 110 && lon <= 180) {
        region = 'oceania';
    }
    // South America
    else if (lat >= -56 && lat <= 13 && lon >= -82 && lon <= -34) {
        region = 'south_america';
    }
    // Africa
    else if (lat >= -35 && lat <= 38 && lon >= -18 && lon <= 52) {
        region = 'africa';
    }

    emit(region);
}

```

---

## 📈 Performance and Analytics Fields

### Statistical Calculations

**Performance metrics and ratios:**

```java
// Calculate conversion rate
if (doc['total_visitors'].size() > 0 && doc['conversions'].size() > 0) {
    double visitors = doc['total_visitors'].value;
    double conversions = doc['conversions'].value;

    if (visitors > 0) {
        double conversionRate = (conversions / visitors) * 100;
        emit(Math.round(conversionRate * 100.0) / 100.0);
    } else {
        emit(0.0);
    }
}

```

**Trend analysis:**

```java
// Calculate growth rate
if (doc['current_value'].size() > 0 && doc['previous_value'].size() > 0) {
    double current = doc['current_value'].value;
    double previous = doc['previous_value'].value;

    if (previous != 0) {
        double growthRate = ((current - previous) / previous) * 100;
        emit(Math.round(growthRate * 100.0) / 100.0);
    } else if (current > 0) {
        emit(100.0); // 100% growth from zero
    } else {
        emit(0.0);
    }
}

```

### Complex Scoring Algorithms

**Multi-factor ranking score:**

```java
// Calculate content quality score
double qualityScore = 0;

// Engagement metrics (40% weight)
if (doc['view_count'].size() > 0) {
    double views = doc['view_count'].value;
    qualityScore += Math.log1p(views) * 0.4;
}

// Recency factor (30% weight)
if (doc['publish_date'].size() > 0) {
    long publishMillis = doc['publish_date'].value.millis;
    long nowMillis = System.currentTimeMillis();
    long ageDays = (nowMillis - publishMillis) / (1000L * 60 * 60 * 24);

    // Exponential decay with 30-day half-life
    double recencyScore = Math.exp(-0.693 * ageDays / 30.0);
    qualityScore += recencyScore * 30;
}

// User ratings (20% weight)
if (doc['average_rating'].size() > 0) {
    double rating = doc['average_rating'].value;
    qualityScore += (rating / 5.0) * 20;
}

// Authority score (10% weight)
if (doc['author_score'].size() > 0) {
    double authorScore = doc['author_score'].value;
    qualityScore += (authorScore / 100.0) * 10;
}

emit(Math.round(qualityScore * 100.0) / 100.0);

```

---

## 🔧 Advanced Runtime Field Techniques

### Conditional Logic and State Machines

**Multi-step conditional processing:**

```java
// Customer lifecycle stage determination
String stage = 'prospect';

if (doc['first_purchase_date'].size() > 0) {
    long firstPurchase = doc['first_purchase_date'].value.millis;
    long lastPurchase = doc['last_purchase_date'].value.millis;
    long nowMillis = System.currentTimeMillis();

    int totalOrders = doc['total_orders'].value;
    double totalSpent = doc['total_spent'].value;
    long daysSinceFirst = (nowMillis - firstPurchase) / (1000L * 60 * 60 * 24);
    long daysSinceLast = (nowMillis - lastPurchase) / (1000L * 60 * 60 * 24);

    if (daysSinceLast > 365) {
        stage = 'churned';
    } else if (daysSinceLast > 180) {
        if (totalSpent > 1000) {
            stage = 'at_risk_vip';
        } else {
            stage = 'at_risk';
        }
    } else if (totalOrders == 1) {
        if (daysSinceFirst <= 30) {
            stage = 'new_customer';
        } else {
            stage = 'one_time_buyer';
        }
    } else if (totalOrders >= 10 || totalSpent >= 2000) {
        stage = 'vip';
    } else if (totalOrders >= 5 || totalSpent >= 500) {
        stage = 'loyal';
    } else {
        stage = 'regular';
    }
}

emit(stage);

```

### Data Quality and Validation

**Data completeness scoring:**

```java
// Calculate profile completeness score
int totalFields = 10; // Total possible fields
int completedFields = 0;

// Check required fields
String[] requiredFields = ['name', 'email', 'phone', 'address', 'birth_date'];
for (String field : requiredFields) {
    if (doc[field].size() > 0 && !doc[field].value.toString().isEmpty()) {
        completedFields++;
    }
}

// Check optional fields
String[] optionalFields = ['company', 'website', 'linkedin', 'twitter', 'bio'];
for (String field : optionalFields) {
    if (doc[field].size() > 0 && !doc[field].value.toString().isEmpty()) {
        completedFields++;
    }
}

double completenessPercentage = (completedFields / (double) totalFields) * 100;
emit(Math.round(completenessPercentage));

```

### Cross-Field Validation

**Business rule validation:**

```java
// Validate business rules and flag inconsistencies
List<String> violations = new ArrayList();

// Price validation
if (doc['sale_price'].size() > 0 && doc['regular_price'].size() > 0) {
    double salePrice = doc['sale_price'].value;
    double regularPrice = doc['regular_price'].value;

    if (salePrice > regularPrice) {
        violations.add('sale_price_higher_than_regular');
    }

    if (salePrice <= 0) {
        violations.add('invalid_sale_price');
    }
}

// Inventory validation
if (doc['stock_quantity'].size() > 0 && doc['available_for_sale'].size() > 0) {
    int stock = doc['stock_quantity'].value;
    boolean available = doc['available_for_sale'].value;

    if (stock <= 0 && available) {
        violations.add('available_with_no_stock');
    }

    if (stock > 0 && !available) {
        violations.add('unavailable_with_stock');
    }
}

// Date validation
if (doc['start_date'].size() > 0 && doc['end_date'].size() > 0) {
    long startMillis = doc['start_date'].value.millis;
    long endMillis = doc['end_date'].value.millis;

    if (endMillis <= startMillis) {
        violations.add('end_date_before_start_date');
    }
}

emit(violations.isEmpty() ? 'valid' : String.join(',', violations));

```

---

## ⚡ Performance Optimization for Runtime Fields

### Caching and Efficiency

**Optimize field access patterns:**

```java
// ✅ Good: Cache field values to avoid repeated access
double price = doc['price'].size() > 0 ? doc['price'].value : 0.0;
double cost = doc['cost'].size() > 0 ? doc['cost'].value : 0.0;
String category = doc['category'].size() > 0 ? doc['category'].value : 'unknown';

// Use cached values in calculations
double margin = price - cost;
double marginPercentage = price > 0 ? (margin / price) * 100 : 0;

// Category-specific logic using cached value
if (category.equals('electronics')) {
    marginPercentage *= 1.1; // Electronics have different margin expectations
}

emit(Math.round(marginPercentage * 100.0) / 100.0);

```

**Avoid expensive operations:**

```java
// ❌ Avoid: Expensive regex operations
// if (doc['description'].value.matches(".*complex.*regex.*pattern.*")) {

// ✅ Better: Use simple string operations
String description = doc['description'].size() > 0 ? doc['description'].value : '';
if (description.contains('keyword') || description.contains('alternative')) {
    emit('matching');
} else {
    emit('non_matching');
}

```

### Query-Time vs Index-Time Trade-offs

**Decision framework for runtime vs indexed fields:**

| Factor | Favor Runtime Fields | Favor Indexed Fields |
| --- | --- | --- |
| **Query frequency** | Low to medium | High frequency |
| **Calculation complexity** | Simple to medium | Any complexity |
| **Data change frequency** | High (values change often) | Low (stable calculations) |
| **Storage constraints** | Storage is expensive | Storage is cheap |
| **Real-time requirements** | Values must be current | Historical snapshots OK |

---

## 🔍 Runtime Fields in Aggregations

### Dynamic Bucketing

**Create custom aggregation buckets:**

```json
{
  "runtime_mappings": {
    "revenue_tier": {
      "type": "keyword",
      "script": {
        "source": """
          double revenue = doc['monthly_revenue'].value;
          if (revenue < 10000) {
            emit('startup');
          } else if (revenue < 100000) {
            emit('small_business');
          } else if (revenue < 1000000) {
            emit('medium_business');
          } else {
            emit('enterprise');
          }
        """
      }
    }
  },
  "aggs": {
    "by_revenue_tier": {
      "terms": {
        "field": "revenue_tier"
      },
      "aggs": {
        "avg_profit_margin": {
          "avg": {"field": "profit_margin"}
        }
      }
    }
  }
}

```

### Calculated Metrics

**Runtime fields in metric aggregations:**

```json
{
  "runtime_mappings": {
    "efficiency_ratio": {
      "type": "double",
      "script": {
        "source": """
          if (doc['output'].size() > 0 && doc['input'].size() > 0) {
            double output = doc['output'].value;
            double input = doc['input'].value;
            emit(input > 0 ? output / input : 0);
          }
        """
      }
    }
  },
  "aggs": {
    "efficiency_stats": {
      "stats": {"field": "efficiency_ratio"}
    },
    "efficiency_percentiles": {
      "percentiles": {
        "field": "efficiency_ratio",
        "percents": [25, 50, 75, 90, 95]
      }
    }
  }
}

```

---

## 💡 Key Takeaways

✅ **Runtime fields enable schema flexibility without reindexing—perfect for evolving requirements**

✅ **Use runtime fields for exploratory analysis and complex calculations**

✅ **Balance performance vs flexibility—frequent operations may benefit from indexed fields**

✅ **Cache field values in scripts to avoid repeated doc access**

✅ **Runtime fields excel in aggregations and analytical workloads**

✅ **Consider query-time vs index-time trade-offs based on usage patterns**

✅ **Test runtime field performance with realistic data volumes and query patterns**

---