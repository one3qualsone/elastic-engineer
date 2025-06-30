# Common Scripting Patterns

## Proven Painless Solutions for Real-World Problems

This guide covers battle-tested Painless patterns that solve common search and analytics challenges. Rather than learning syntax in isolation, you'll see complete solutions to problems you'll encounter in production, from custom relevance scoring to complex data transformations.

---

## 🎯 Custom Relevance Scoring Patterns

### E-commerce Product Scoring

**Business requirement:** Rank products by relevance, but boost based on inventory, ratings, sales, and business priorities.

```java
// Multi-factor product scoring
double baseScore = _score;

// Get product data
double price = doc['price'].value;
double rating = doc['rating'].empty ? 3.0 : doc['rating'].value;
int stockLevel = doc['stock_level'].value;
boolean isFeatured = doc['is_featured'].value;
long salesCount = doc['sales_count'].value;

// Inventory availability boost
double inventoryBoost = 1.0;
if (stockLevel > 50) {
    inventoryBoost = 1.2;  // High stock
} else if (stockLevel > 10) {
    inventoryBoost = 1.0;  // Normal stock
} else if (stockLevel > 0) {
    inventoryBoost = 0.8;  // Low stock
} else {
    return 0;  // Out of stock - hide completely
}

// Quality boost based on ratings
double qualityBoost = Math.pow(rating / 5.0, 2);  // Non-linear rating boost

// Popularity boost (logarithmic to prevent outliers dominating)
double popularityBoost = Math.log1p(salesCount) / 10.0;

// Price competitiveness (closer to user's target price = better)
double targetPrice = params.user_price_preference;
double priceDistance = Math.abs(price - targetPrice);
double priceBoost = 1.0 / (1.0 + priceDistance / targetPrice);

// Business boost for featured products
double businessBoost = isFeatured ? 1.5 : 1.0;

// Combine all factors
double finalScore = baseScore * inventoryBoost * (1.0 + qualityBoost + popularityBoost + priceBoost) * businessBoost;

return Math.max(finalScore, 0.001);  // Ensure positive score

```

### Content Freshness and Authority Scoring

**Business requirement:** Balance content relevance with recency and author authority.

```java
// Content authority and freshness scoring
double baseScore = _score;

// Time-based scoring
long nowMillis = System.currentTimeMillis();
long publishMillis = doc['publish_date'].value.millis;
long ageHours = (nowMillis - publishMillis) / (1000 * 60 * 60);

// Exponential decay for freshness (half-life approach)
double halfLifeHours = params.content_half_life_hours;  // e.g., 168 (1 week)
double freshnessScore = Math.exp(-0.693 * ageHours / halfLifeHours);

// Author authority
int authorScore = doc['author_score'].empty ? 50 : doc['author_score'].value;
double authorityBoost = (authorScore / 100.0) + 0.5;  // 0.5-1.5 range

// Content engagement
int viewCount = doc['view_count'].value;
int shareCount = doc['share_count'].value;
double engagementScore = Math.log1p(viewCount + shareCount * 5) / 15.0;

// Content type preference
String contentType = doc['content_type'].value;
Map<String, Double> typeBoosts = params.content_type_boosts;
double typeBoost = typeBoosts.getOrDefault(contentType, 1.0);

return baseScore * (1.0 + freshnessScore + engagementScore) * authorityBoost * typeBoost;

```

### Geographic Relevance Scoring

**Business requirement:** Boost local results while maintaining global relevance.

```java
// Geographic proximity scoring
double baseScore = _score;

// User location (passed via params)
double userLat = params.user_latitude;
double userLon = params.user_longitude;

// Business location
if (doc['location'].empty) {
    return baseScore;  // No location data, use base score
}

double bizLat = doc['location'].lat;
double bizLon = doc['location'].lon;

// Calculate distance using Haversine formula (simplified)
double dLat = Math.toRadians(bizLat - userLat);
double dLon = Math.toRadians(bizLon - userLon);
double a = Math.sin(dLat/2) * Math.sin(dLat/2) +
           Math.cos(Math.toRadians(userLat)) * Math.cos(Math.toRadians(bizLat)) *
           Math.sin(dLon/2) * Math.sin(dLon/2);
double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
double distance = 6371 * c;  // Distance in kilometers

// Distance-based boost (closer = higher score)
double proximityBoost = 1.0;
if (distance <= 5) {
    proximityBoost = 2.0;      // Very close (5km)
} else if (distance <= 25) {
    proximityBoost = 1.5;      // Close (25km)
} else if (distance <= 100) {
    proximityBoost = 1.2;      // Nearby (100km)
} else {
    proximityBoost = 1.0;      // Distant
}

// Local business preference
boolean isLocalBusiness = doc['is_local_business'].value;
double localBoost = isLocalBusiness ? 1.3 : 1.0;

return baseScore * proximityBoost * localBoost;

```

---

## 🔄 Data Transformation Patterns

### Data Cleansing and Standardization

**Use case:** Clean and standardize data during indexing via ingest pipelines.

```java
// Email standardization
String email = ctx.email;
if (email != null) {
    // Convert to lowercase and trim
    email = email.toLowerCase().trim();

    // Remove extra spaces
    email = email.replaceAll('\\s+', '');

    // Basic validation
    if (email.contains('@') && email.contains('.')) {
        ctx.email_clean = email;
        ctx.email_domain = email.substring(email.indexOf('@') + 1);
    } else {
        ctx.email_clean = null;
        ctx.email_domain = null;
    }
}

// Phone number standardization
String phone = ctx.phone_number;
if (phone != null) {
    // Remove all non-numeric characters
    String cleanPhone = phone.replaceAll('[^0-9]', '');

    // Format based on length
    if (cleanPhone.length() == 10) {
        ctx.phone_clean = cleanPhone;
        ctx.phone_formatted = cleanPhone.substring(0, 3) + '-' +
                             cleanPhone.substring(3, 6) + '-' +
                             cleanPhone.substring(6);
    } else if (cleanPhone.length() == 11 && cleanPhone.startsWith('1')) {
        ctx.phone_clean = cleanPhone.substring(1);  // Remove country code
        ctx.phone_formatted = cleanPhone.substring(1, 4) + '-' +
                             cleanPhone.substring(4, 7) + '-' +
                             cleanPhone.substring(7);
    } else {
        ctx.phone_clean = cleanPhone;
        ctx.phone_formatted = phone;  // Keep original if can't parse
    }
}

// Name standardization
String fullName = ctx.full_name;
if (fullName != null) {
    String[] nameParts = fullName.trim().split('\\s+');
    if (nameParts.length >= 2) {
        ctx.first_name = nameParts[0];
        ctx.last_name = nameParts[nameParts.length - 1];

        if (nameParts.length > 2) {
            // Join middle names
            StringBuilder middle = new StringBuilder();
            for (int i = 1; i < nameParts.length - 1; i++) {
                if (middle.length() > 0) middle.append(' ');
                middle.append(nameParts[i]);
            }
            ctx.middle_name = middle.toString();
        }
    }
}

```

### Category and Tag Processing

**Use case:** Process and enrich categorical data with hierarchies and related terms.

```java
// Category hierarchy processing
List<String> categories = ctx.categories;
if (categories != null && !categories.isEmpty()) {
    List<String> processedCategories = new ArrayList();
    List<String> categoryHierarchy = new ArrayList();
    Set<String> allCategories = new HashSet();

    // Process each category
    for (String category : categories) {
        String cleanCategory = category.toLowerCase().trim();
        processedCategories.add(cleanCategory);
        allCategories.add(cleanCategory);

        // Build hierarchy path
        String[] parts = cleanCategory.split('/');
        StringBuilder path = new StringBuilder();
        for (String part : parts) {
            if (path.length() > 0) path.append('/');
            path.append(part);
            categoryHierarchy.add(path.toString());
            allCategories.add(path.toString());
        }

        // Add parent categories
        if (parts.length > 1) {
            allCategories.add(parts[0]);  // Top level
        }
    }

    ctx.categories_processed = new ArrayList(processedCategories);
    ctx.category_hierarchy = new ArrayList(categoryHierarchy);
    ctx.all_categories = new ArrayList(allCategories);

    // Primary category (first one)
    ctx.primary_category = processedCategories.get(0);

    // Category level
    ctx.category_depth = ctx.primary_category.split('/').length;
}

// Tag enhancement and normalization
List<String> tags = ctx.tags;
if (tags != null && !tags.isEmpty()) {
    Set<String> enhancedTags = new HashSet();
    Map<String, List<String>> tagSynonyms = params.tag_synonyms;

    for (String tag : tags) {
        String cleanTag = tag.toLowerCase().trim().replaceAll('[^a-z0-9]', '');
        enhancedTags.add(cleanTag);

        // Add synonyms
        if (tagSynonyms.containsKey(cleanTag)) {
            enhancedTags.addAll(tagSynonyms.get(cleanTag));
        }

        // Add variations
        if (cleanTag.endsWith('s') && cleanTag.length() > 3) {
            enhancedTags.add(cleanTag.substring(0, cleanTag.length() - 1));  // Singular
        }
    }

    ctx.tags_enhanced = new ArrayList(enhancedTags);
    ctx.tag_count = enhancedTags.size();
}

```

### Price and Currency Processing

**Use case:** Normalize prices across different currencies and handle price ranges.

```java
// Price normalization and analysis
Object priceObj = ctx.price;
String currency = ctx.currency;

if (priceObj != null) {
    double price = ((Number) priceObj).doubleValue();

    // Convert to USD for comparison
    Map<String, Double> exchangeRates = params.exchange_rates;
    double usdPrice = price;

    if (currency != null && !currency.equals('USD')) {
        Double rate = exchangeRates.get(currency);
        if (rate != null) {
            usdPrice = price * rate;
        }
    }

    ctx.price_usd = Math.round(usdPrice * 100.0) / 100.0;  // Round to 2 decimals

    // Price categorization
    if (usdPrice < 25) {
        ctx.price_category = 'budget';
    } else if (usdPrice < 100) {
        ctx.price_category = 'mid-range';
    } else if (usdPrice < 500) {
        ctx.price_category = 'premium';
    } else {
        ctx.price_category = 'luxury';
    }

    // Price tier (for faceted search)
    if (usdPrice < 50) {
        ctx.price_tier = 'under-50';
    } else if (usdPrice < 100) {
        ctx.price_tier = '50-100';
    } else if (usdPrice < 250) {
        ctx.price_tier = '100-250';
    } else if (usdPrice < 500) {
        ctx.price_tier = '250-500';
    } else {
        ctx.price_tier = 'over-500';
    }
}

// Handle price ranges
Object salePriceObj = ctx.sale_price;
if (salePriceObj != null && priceObj != null) {
    double regularPrice = ((Number) priceObj).doubleValue();
    double salePrice = ((Number) salePriceObj).doubleValue();

    if (salePrice < regularPrice) {
        ctx.on_sale = true;
        ctx.discount_amount = regularPrice - salePrice;
        ctx.discount_percentage = Math.round(((regularPrice - salePrice) / regularPrice) * 100);
        ctx.effective_price = salePrice;
    } else {
        ctx.on_sale = false;
        ctx.effective_price = regularPrice;
    }
}

```

---

## 📊 Aggregation and Analytics Patterns

### Custom Bucket Aggregations

**Use case:** Create custom groupings that don't map directly to field values.

```java
// Custom time period bucketing
long timestamp = doc['@timestamp'].value.millis;
Calendar cal = Calendar.getInstance();
cal.setTimeInMillis(timestamp);

int hour = cal.get(Calendar.HOUR_OF_DAY);
int dayOfWeek = cal.get(Calendar.DAY_OF_WEEK);

// Business hours classification
String timePeriod;
if (dayOfWeek >= 2 && dayOfWeek <= 6) {  // Monday-Friday
    if (hour >= 9 && hour <= 17) {
        timePeriod = 'business-hours';
    } else if (hour >= 18 && hour <= 22) {
        timePeriod = 'evening';
    } else {
        timePeriod = 'night';
    }
} else {  // Weekend
    if (hour >= 10 && hour <= 18) {
        timePeriod = 'weekend-day';
    } else {
        timePeriod = 'weekend-night';
    }
}

return timePeriod;

```

### Calculated Metrics

**Use case:** Compute complex metrics that combine multiple fields.

```java
// Customer lifetime value calculation
double totalOrderValue = doc['total_order_value'].value;
int orderCount = doc['order_count'].value;
long firstOrderDate = doc['first_order_date'].value.millis;
long lastOrderDate = doc['last_order_date'].value.millis;

// Customer tenure in days
long tenureDays = (lastOrderDate - firstOrderDate) / (1000 * 60 * 60 * 24);

// Average order value
double avgOrderValue = orderCount > 0 ? totalOrderValue / orderCount : 0;

// Order frequency (orders per month)
double orderFrequency = tenureDays > 0 ?
    (orderCount * 30.0) / tenureDays : 0;

// Customer lifetime value score
double clvScore = avgOrderValue * orderFrequency * Math.sqrt(tenureDays / 30.0);

return Math.max(clvScore, 0);

```

### Percentile and Statistical Calculations

**Use case:** Calculate percentiles and statistical measures within aggregations.

```java
// Performance scoring based on percentiles
List<Double> values = new ArrayList();

// Collect values from related documents (in aggregation context)
for (def val : doc['response_times']) {
    values.add(val);
}

if (values.isEmpty()) {
    return 0;
}

// Sort values for percentile calculation
Collections.sort(values);
int size = values.size();

// Calculate percentiles
double p50 = values.get((int)(size * 0.5));
double p95 = values.get((int)(size * 0.95));
double p99 = values.get((int)(size * 0.99));

// Performance score based on percentile thresholds
double score = 100;  // Start with perfect score

if (p50 > 100) score -= 10;      // P50 over 100ms
if (p95 > 500) score -= 20;      // P95 over 500ms
if (p99 > 1000) score -= 30;     // P99 over 1s

return Math.max(score, 0);

```

---

## 🔄 Update and Upsert Patterns

### Safe Counter Updates

**Use case:** Safely increment counters and maintain statistics.

```java
// Safe view counter with session tracking
String sessionId = params.session_id;
String userId = params.user_id;

// Initialize counters if they don't exist
if (ctx._source.view_count == null) {
    ctx._source.view_count = 0;
}

if (ctx._source.unique_viewers == null) {
    ctx._source.unique_viewers = new HashSet();
}

if (ctx._source.sessions_today == null) {
    ctx._source.sessions_today = new HashMap();
}

// Get current date for daily tracking
String today = new SimpleDateFormat('yyyy-MM-dd').format(new Date());

// Always increment view count
ctx._source.view_count++;

// Track unique viewers
if (userId != null) {
    ctx._source.unique_viewers.add(userId);
}

// Track daily sessions
if (!ctx._source.sessions_today.containsKey(today)) {
    ctx._source.sessions_today.put(today, new HashSet());
}
ctx._source.sessions_today.get(today).add(sessionId);

// Update last viewed timestamp
ctx._source.last_viewed = new Date();

// Calculate engagement metrics
int totalViews = ctx._source.view_count;
int uniqueViewers = ctx._source.unique_viewers.size();
double engagementRatio = uniqueViewers > 0 ? (double)totalViews / uniqueViewers : 0;

ctx._source.engagement_ratio = Math.round(engagementRatio * 100.0) / 100.0;

```

### Conditional Document Updates

**Use case:** Update documents only when certain conditions are met.

```java
// Update product pricing with business rules
double newPrice = params.new_price;
double currentPrice = ctx._source.price;
boolean isOnSale = ctx._source.on_sale;
String updateReason = params.reason;

// Business rules for price updates
boolean canUpdate = false;
String updateStatus = 'rejected';

if (updateReason.equals('competitor_match')) {
    // Allow price drops for competitor matching
    if (newPrice < currentPrice) {
        canUpdate = true;
        updateStatus = 'approved_competitor_match';
    }
} else if (updateReason.equals('inventory_clearance')) {
    // Allow significant price drops for clearance
    if (newPrice < currentPrice * 0.7) {  // At least 30% off
        canUpdate = true;
        updateStatus = 'approved_clearance';
    }
} else if (updateReason.equals('market_adjustment')) {
    // Allow price increases up to 10%
    if (newPrice <= currentPrice * 1.1) {
        canUpdate = true;
        updateStatus = 'approved_market_adjustment';
    }
}

if (canUpdate) {
    // Update price and related fields
    ctx._source.price = newPrice;
    ctx._source.price_updated = new Date();
    ctx._source.update_reason = updateReason;

    // Update sale status
    if (newPrice < ctx._source.original_price) {
        ctx._source.on_sale = true;
        ctx._source.discount_percentage =
            Math.round(((ctx._source.original_price - newPrice) /
                       ctx._source.original_price) * 100);
    } else {
        ctx._source.on_sale = false;
        ctx._source.discount_percentage = 0;
    }

    // Add to price history
    if (ctx._source.price_history == null) {
        ctx._source.price_history = new ArrayList();
    }

    ctx._source.price_history.add([
        'price': currentPrice,
        'date': new Date(),
        'reason': 'replaced_by_' + updateReason
    ]);

} else {
    // Don't update the document
    ctx.op = 'none';
}

// Always log the attempt
if (ctx._source.update_attempts == null) {
    ctx._source.update_attempts = new ArrayList();
}

ctx._source.update_attempts.add([
    'attempted_price': newPrice,
    'current_price': currentPrice,
    'reason': updateReason,
    'status': updateStatus,
    'timestamp': new Date()
]);

```

### Merging Nested Objects

**Use case:** Merge new data into existing nested structures.

```java
// Merge user profile data
Map<String, Object> newProfile = params.profile_updates;
Map<String, Object> currentProfile = ctx._source.profile;

if (currentProfile == null) {
    currentProfile = new HashMap();
    ctx._source.profile = currentProfile;
}

// Merge preferences
if (newProfile.containsKey('preferences')) {
    Map<String, Object> newPrefs = (Map<String, Object>) newProfile.preferences;
    Map<String, Object> currentPrefs = (Map<String, Object>) currentProfile.preferences;

    if (currentPrefs == null) {
        currentPrefs = new HashMap();
        currentProfile.preferences = currentPrefs;
    }

    // Merge preference categories
    for (String prefCategory : newPrefs.keySet()) {
        if (currentPrefs.containsKey(prefCategory) &&
            currentPrefs.get(prefCategory) instanceof Map) {
            // Merge existing category
            Map<String, Object> currentCategoryPrefs =
                (Map<String, Object>) currentPrefs.get(prefCategory);
            Map<String, Object> newCategoryPrefs =
                (Map<String, Object>) newPrefs.get(prefCategory);

            for (String prefKey : newCategoryPrefs.keySet()) {
                currentCategoryPrefs.put(prefKey, newCategoryPrefs.get(prefKey));
            }
        } else {
            // Replace entire category
            currentPrefs.put(prefCategory, newPrefs.get(prefCategory));
        }
    }
}

// Update metadata
currentProfile.put('last_updated', new Date());
currentProfile.put('update_count',
    ((Integer) currentProfile.getOrDefault('update_count', 0)) + 1);

```

---

## 🛠️ Integration and Data Processing Patterns

### API Response Processing

**Use case:** Process and normalize data from external APIs during ingest.

```java
// Process weather API response
Map<String, Object> weatherData = ctx.weather_api_response;

if (weatherData != null) {
    // Extract current conditions
    Map<String, Object> current = (Map<String, Object>) weatherData.current;
    if (current != null) {
        ctx.temperature_celsius = current.temp_c;
        ctx.temperature_fahrenheit = current.temp_f;
        ctx.humidity = current.humidity;
        ctx.wind_speed = current.wind_kph;
        ctx.weather_condition = ((String) current.condition.text).toLowerCase();
    }

    // Extract forecast
    List<Map<String, Object>> forecast =
        (List<Map<String, Object>>) weatherData.forecast.forecastday;

    if (forecast != null && !forecast.isEmpty()) {
        List<Map<String, Object>> processedForecast = new ArrayList();

        for (Map<String, Object> day : forecast) {
            Map<String, Object> dayData = (Map<String, Object>) day.day;

            processedForecast.add([
                'date': day.date,
                'max_temp': dayData.maxtemp_c,
                'min_temp': dayData.mintemp_c,
                'condition': dayData.condition.text,
                'chance_of_rain': dayData.chance_of_rain
            ]);
        }

        ctx.weather_forecast = processedForecast;
    }

    // Calculate weather score for outdoor activities
    double weatherScore = 0;
    if (current != null) {
        double temp = (Double) current.temp_c;
        int humidity = (Integer) current.humidity;
        String condition = (String) current.condition.text;

        // Temperature score (optimal around 20-25°C)
        if (temp >= 20 && temp <= 25) {
            weatherScore += 40;
        } else if (temp >= 15 && temp <= 30) {
            weatherScore += 30;
        } else if (temp >= 10 && temp <= 35) {
            weatherScore += 20;
        }

        // Humidity score (lower is better)
        if (humidity <= 60) {
            weatherScore += 30;
        } else if (humidity <= 80) {
            weatherScore += 20;
        }

        // Condition score
        if (condition.contains('sunny') || condition.contains('clear')) {
            weatherScore += 30;
        } else if (condition.contains('cloudy')) {
            weatherScore += 20;
        } else if (condition.contains('rain')) {
            weatherScore += 5;
        }
    }

    ctx.weather_score = weatherScore;

    // Remove raw API response to save space
    ctx.remove('weather_api_response');
}

```

### Log Parsing and Enrichment

**Use case:** Parse log entries and extract structured data.

```java
// Parse Apache access log format
String logLine = ctx.message;

if (logLine != null) {
    // Parse common log format: IP - - [timestamp] "method URL protocol" status size
    String pattern = '^(\\S+) (\\S+) (\\S+) \\[([^\\]]+)\\] "([^"]+)" (\\d+) (\\S+)';

    // Simple parsing (in real scenarios, use Grok processor)
    String[] parts = logLine.split(' ');

    if (parts.length >= 7) {
        ctx.client_ip = parts[0];

        // Parse request line
        if (parts.length > 5 && parts[5].startsWith('"')) {
            String requestLine = logLine.substring(logLine.indexOf('"') + 1,
                                                 logLine.lastIndexOf('"'));
            String[] requestParts = requestLine.split(' ');

            if (requestParts.length >= 3) {
                ctx.http_method = requestParts[0];
                ctx.request_url = requestParts[1];
                ctx.http_version = requestParts[2];

                // Parse URL components
                String url = requestParts[1];
                if (url.contains('?')) {
                    ctx.url_path = url.substring(0, url.indexOf('?'));
                    ctx.url_query = url.substring(url.indexOf('?') + 1);
                } else {
                    ctx.url_path = url;
                }

                // Extract file extension
                String path = ctx.url_path;
                if (path.contains('.')) {
                    ctx.file_extension = path.substring(path.lastIndexOf('.') + 1);
                }
            }
        }

        // Parse status code and size
        ctx.status_code = Integer.parseInt(parts[parts.length - 2]);
        String sizeStr = parts[parts.length - 1];
        ctx.response_size = sizeStr.equals('-') ? 0 : Integer.parseInt(sizeStr);
    }

    // Categorize request types
    String method = ctx.http_method;
    String path = ctx.url_path;

    if (method != null && path != null) {
        if (path.startsWith('/api/')) {
            ctx.request_type = 'api';
        } else if (path.startsWith('/admin/')) {
            ctx.request_type = 'admin';
        } else if (path.endsWith('.css') || path.endsWith('.js') ||
                   path.endsWith('.png') || path.endsWith('.jpg')) {
            ctx.request_type = 'static';
        } else {
            ctx.request_type = 'page';
        }
    }

    // Determine response category
    int status = ctx.status_code;
    if (status >= 200 && status < 300) {
        ctx.response_category = 'success';
    } else if (status >= 300 && status < 400) {
        ctx.response_category = 'redirect';
    } else if (status >= 400 && status < 500) {
        ctx.response_category = 'client_error';
    } else if (status >= 500) {
        ctx.response_category = 'server_error';
    }
}

```

---

## 💡 Key Takeaways

✅ **Combine multiple factors in scoring scripts to reflect real business requirements**

✅ **Use early returns and guard clauses to optimize script performance**

✅ **Pre-process and clean data during ingestion to avoid runtime complexity**

✅ **Handle edge cases gracefully with null checks and default values**

✅ **Cache expensive calculations and reuse them across script contexts**

✅ **Structure complex scripts with clear variable names and logical sections**

✅ **Test scripts with realistic data to ensure they behave as expected in production**

---