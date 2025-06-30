# Systematic Query Reasoning

## A Step-by-Step Method for Building Any Elasticsearch Query

Instead of memorizing Query DSL syntax, let's learn a systematic approach to reasoning through any search problem. This method will help you construct queries logically, even when facing unfamiliar requirements.

---

## 🎯 The Query Reasoning Framework

When faced with any search requirement, follow this systematic process:

| Step | Question | Purpose |
| --- | --- | --- |
| **1. Intent** | What is the user trying to accomplish? | Understand the real goal |
| **2. Must Have** | What absolutely must be true? | Identify required conditions |
| **3. Nice to Have** | What would make results better? | Identify scoring boosts |
| **4. Must Not Have** | What should be excluded? | Identify exclusions |
| **5. Context Type** | Is this about relevance or filtering? | Choose query vs filter context |

Let's apply this framework to real scenarios.

---

## 🔍 Example 1: E-commerce Search

**User Request:** "Show me wireless headphones under $100 that are in stock, preferably from Sony"

### Step 1: Understand Intent

**What they're really asking:** Find products that match their search terms, within budget, available for purchase, with brand preference.

### Step 2: Identify Must-Have Conditions

| Condition | Type | Reasoning |
| --- | --- | --- |
| Contains "wireless" and "headphones" | **Query context** | Text search needs relevance scoring |
| Price < $100 | **Filter context** | Exact boundary, no scoring needed |
| In stock = true | **Filter context** | Binary condition, no scoring needed |

### Step 3: Identify Nice-to-Have Conditions

| Condition | Type | Reasoning |
| --- | --- | --- |
| Brand = "Sony" | **Should clause** | Preference, not requirement |

### Step 4: Identify Exclusions

**None in this case** - no explicit exclusions mentioned.

### Step 5: Choose Structure

**Result:** Bool query with must (text search) + filter (exact conditions) + should (preferences)

---

## 🔧 The Systematic Building Process

### Phase 1: Start with Core Requirements

**Question:** What's the absolutely essential thing this query must accomplish?

**In our example:** Find products matching "wireless headphones"

**Basic structure:**

- One `match` or `multi_match` query for text search
- This goes in `must` clause because it's required

### Phase 2: Add Exact Conditions

**Question:** What are the non-negotiable boundaries or filters?

**In our example:** Price limit and stock status

**Add to structure:**

- Range query for price → `filter` clause
- Term query for stock status → `filter` clause

### Phase 3: Add Preferences

**Question:** What would make results better but isn't required?

**In our example:** Sony brand preference

**Add to structure:**

- Term query for brand → `should` clause

### Phase 4: Add Exclusions

**Question:** What should definitely not appear in results?

**In our example:** Nothing specific

**Would add to structure:** `must_not` clause (if needed)

---

## 🧠 Decision Framework: Query vs Filter Context

Use this decision tree for every condition:

```
Does this condition affect how relevant a document is?
├── YES → Query context (must/should clauses)
│   ├── Required for relevance → must clause
│   └── Nice for relevance → should clause
└── NO → Filter context (filter/must_not clauses)
    ├── Must be true → filter clause
    └── Must be false → must_not clause

```

### Context Decision Examples

| Condition | Decision Process | Result |
| --- | --- | --- |
| "wireless headphones" | Affects relevance (some products more relevant) | Query context |
| price < $100 | Binary boundary (in budget or not) | Filter context |
| category = "electronics" | Exact match requirement | Filter context |
| published_date > "2024-01-01" | Exact date boundary | Filter context |
| content contains "tutorial" | Affects relevance (some more tutorial-like) | Query context |
| status != "draft" | Binary exclusion | Filter context |

---

## 🎨 Common Query Patterns by Intent

### Pattern 1: User Search Box

**Intent:** User types text, wants relevant results

**Systematic approach:**

1. **Core requirement:** Text search with relevance
2. **Applied filters:** Category, price range, availability
3. **Boosts:** Brand preferences, ratings
4. **Exclusions:** Out of stock, discontinued

**Structure:** Multi-match (must) + filters + should boosts

### Pattern 2: Dashboard Filter

**Intent:** Show data matching specific criteria

**Systematic approach:**

1. **Core requirement:** Time range
2. **Applied filters:** Service names, error types
3. **Boosts:** None needed
4. **Exclusions:** Test data, internal traffic

**Structure:** All filter context (no relevance scoring needed)

### Pattern 3: Content Discovery

**Intent:** Find related content user might like

**Systematic approach:**

1. **Core requirement:** Similar topics or categories
2. **Applied filters:** Published, not already seen
3. **Boosts:** Popularity, recency, user interests
4. **Exclusions:** User's own content

**Structure:** Multiple should clauses with minimum match

---

## 🚫 Common Reasoning Mistakes

### Mistake 1: Using Query Context for Exact Matches

**Wrong reasoning:** "I need documents where status equals 'published'"
**Wrong choice:** `match` query (implies relevance matters)
**Correct reasoning:** "Status is binary - either published or not"

**Correct choice:** `term` filter (exact match, no scoring)

### Mistake 2: Using Filter Context for Text Search

**Wrong reasoning:** "I need documents containing 'elasticsearch'"
**Wrong choice:** `term` filter (looks for exact word "elasticsearch")
**Correct reasoning:** "Users might type variations, synonyms, or related terms"
**Correct choice:** `match` query (handles analysis and relevance)

### Mistake 3: Overcomplicating with Nested Bools

**Wrong approach:** Complex nested boolean logic that's hard to understand
**Better approach:** Break complex requirements into multiple simpler queries
**Alternative approach:** Use application logic to combine multiple simple queries

---

## 📝 The Query Building Worksheet

For any new query requirement, work through this checklist:

### Understanding Phase

- [ ]  What is the user's actual goal?
- [ ]  What information do they need in the response?
- [ ]  How will results be displayed/used?

### Requirements Analysis

- [ ]  What absolutely must be true? (must/filter)
- [ ]  What would make results better? (should)
- [ ]  What should be excluded? (must_not)
- [ ]  Is relevance scoring important for each condition?

### Structure Planning

- [ ]  What's the core query type? (match, term, range, bool)
- [ ]  Which conditions need query context vs filter context?
- [ ]  Do I need multiple query types combined?
- [ ]  How should results be sorted?

### Validation

- [ ]  Does this query answer the user's actual question?
- [ ]  Are all conditions in the right context (query vs filter)?
- [ ]  Is the query as simple as possible while meeting requirements?

---

## 🎯 Practical Application: Working Through a Complex Example

**Scenario:** "Find recent blog posts about machine learning that are suitable for beginners, published by senior engineers, but not just promotional content"

### Apply the Framework:

**Step 1 - Intent Analysis:**

- **Goal:** Educational content discovery
- **Audience:** Beginners learning ML
- **Quality signal:** Author expertise
- **Exclusion:** Marketing content

**Step 2 - Requirement Categorization:**

| Requirement | Must/Should/Must Not | Query/Filter | Reasoning |
| --- | --- | --- | --- |
| Content about "machine learning" | Must | Query | Relevance scoring matters |
| Suitable for beginners | Should | Query | Some posts more beginner-friendly |
| Recent (last 6 months) | Must | Filter | Exact time boundary |
| Author is senior engineer | Should | Query | Improves quality but not required |
| Not promotional content | Must Not | Filter | Binary exclusion |

**Step 3 - Structure Planning:**

- Bool query with multiple clauses
- Text search in must clause
- Date filter in filter clause
- Quality signals in should clauses
- Content type exclusion in must_not clause

---

## 🚀 Building Query Intuition

### The Mental Model

Instead of thinking: *"What Query DSL syntax do I need?"*

Think: *"What logical conditions must be true for a document to be a good result?"*

### The Process

1. **Write out the requirements in plain English**
2. **Categorize each requirement** (must/should/must_not + query/filter)
3. **Map to Query DSL structure** based on categories
4. **Start simple and add complexity** incrementally

### Building Confidence

**Start with simple queries:**

- Single match query
- Single filter
- Combine two conditions

**Gradually increase complexity:**

- Multiple filters
- Mix of query and filter context
- Multiple should clauses

**Test your reasoning:**

- Can you explain why each clause is needed?
- Can you explain why it's in query vs filter context?
- Can you predict what documents would match?

---

## 💡 Key Takeaways

✅ **Use systematic reasoning, not memorization**

✅ **Understand user intent before building queries**

✅ **Categorize requirements by must/should/must_not**

✅ **Choose query vs filter context based on relevance needs**

✅ **Start simple and build complexity gradually**

✅ **Validate that your query answers the actual question**

---