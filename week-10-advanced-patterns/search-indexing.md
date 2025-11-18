# Search & Indexing Systems

## Overview

Search is fundamentally different from database queries. Users type messy, incomplete, misspelled queries and expect relevant results instantly. Building production search requires understanding inverted indexes, relevance scoring, and distributed search architectures.

## The Search Problem

### Database Query (Exact Match)
```sql
SELECT * FROM products WHERE name = 'iPhone 15 Pro';
-- Returns: Nothing (user typed "iphone15pro")
```

### Search Query (Fuzzy Match)
```
Query: "iphone15pro"
Results:
1. iPhone 15 Pro Max (relevance: 0.95)
2. iPhone 15 Pro (relevance: 0.92)
3. iPhone 15 (relevance: 0.75)
```

**Key differences:**
- Tolerates typos, variations
- Ranks by relevance
- Understands synonyms
- Searches across multiple fields
- Sub-second latency even with billions of documents

## Inverted Index (The Core Data Structure)

### How It Works

**Forward Index (like a database):**
```
Doc1: "The quick brown fox"
Doc2: "The lazy brown dog"
Doc3: "Quick brown foxes"
```

**Inverted Index (search engine):**
```
"the"    -> [Doc1, Doc2]
"quick"  -> [Doc1, Doc3]
"brown"  -> [Doc1, Doc2, Doc3]
"fox"    -> [Doc1, Doc3]  (stemming: fox, foxes)
"lazy"   -> [Doc2]
"dog"    -> [Doc2]
```

**Query: "quick fox"**
```
"quick" -> [Doc1, Doc3]
"fox"   -> [Doc1, Doc3]
Intersection -> [Doc1, Doc3]
```

### Building an Inverted Index

```python
from collections import defaultdict
import re

class InvertedIndex:
    def __init__(self):
        self.index = defaultdict(set)  # term -> set of doc_ids

    def add_document(self, doc_id, text):
        tokens = self.tokenize(text)
        for token in tokens:
            self.index[token].add(doc_id)

    def tokenize(self, text):
        # Lowercase, split, remove punctuation
        text = text.lower()
        tokens = re.findall(r'\w+', text)
        return tokens

    def search(self, query):
        tokens = self.tokenize(query)
        if not tokens:
            return set()

        # Get documents containing first token
        results = self.index[tokens[0]].copy()

        # Intersect with documents containing other tokens
        for token in tokens[1:]:
            results &= self.index[token]

        return results

# Usage
index = InvertedIndex()
index.add_document(1, "The quick brown fox")
index.add_document(2, "The lazy brown dog")
index.add_document(3, "Quick brown foxes")

print(index.search("quick fox"))  # {1, 3}
```

## Elasticsearch Architecture

### Cluster Structure

```
Elasticsearch Cluster
│
├── Node 1 (Master-eligible, Data)
│   ├── Index: products
│   │   ├── Shard 0 (Primary)
│   │   └── Shard 1 (Replica)
│   └── Index: users
│       └── Shard 0 (Primary)
│
├── Node 2 (Data)
│   ├── Index: products
│   │   ├── Shard 1 (Primary)
│   │   └── Shard 0 (Replica)
│   └── Index: users
│       └── Shard 1 (Replica)
│
└── Node 3 (Coordinating)
    └── Routes queries, aggregates results
```

### Sharding in Elasticsearch

**Why shard?**
- Index too big for one node
- Parallelize search (search all shards simultaneously)
- Distribute load

**How many shards?**
```
Number of shards = (Total index size / Desired shard size)

Example:
- 100GB index
- 30GB per shard (recommended max 50GB)
- Shards needed: 100 / 30 ≈ 4 shards
```

**Shard allocation:**
```json
PUT /products
{
  "settings": {
    "number_of_shards": 4,
    "number_of_replicas": 1  // Each shard has 1 replica
  }
}
```

### Document Indexing Flow

```
1. Client sends document to any node (coordinating node)
   POST /products/_doc/123
   { "name": "iPhone 15", "price": 999 }

2. Coordinating node routes to primary shard
   hash(doc_id) % num_shards -> Shard 2

3. Primary shard (Node A) indexes document
   - Adds to inverted index
   - Adds to transaction log

4. Primary replicates to replica shards (Node B, Node C)

5. Ack sent back to client
```

### Search Flow

```
1. Client sends query to any node
   GET /products/_search
   { "query": { "match": { "name": "iphone" } } }

2. Coordinating node broadcasts query to all shards
   - Query sent to Shard 0, 1, 2, 3 in parallel

3. Each shard executes query locally
   - Returns top 10 results with scores

4. Coordinating node merges results
   - Global top 10 from all shards' results

5. Coordinating node fetches full documents
   - Only for final top 10

6. Returns to client
```

## Text Analysis & Tokenization

### Analyzer Pipeline

```
Input: "The QUICK Brown Fox's Jumps!"

1. Character Filter: Remove special chars
   -> "The QUICK Brown Foxs Jumps"

2. Tokenizer: Split into tokens
   -> ["The", "QUICK", "Brown", "Foxs", "Jumps"]

3. Token Filters:
   - Lowercase: ["the", "quick", "brown", "foxs", "jumps"]
   - Stop words: ["quick", "brown", "foxs", "jumps"]  (removed "the")
   - Stemmer: ["quick", "brown", "fox", "jump"]  (foxs -> fox, jumps -> jump)

Final tokens: ["quick", "brown", "fox", "jump"]
```

### Elasticsearch Analyzer Example

```json
PUT /products
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_custom_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "stop", "snowball"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "my_custom_analyzer"
      },
      "description": {
        "type": "text",
        "analyzer": "standard"
      }
    }
  }
}
```

### Common Analyzers

| **Analyzer**     | **Use Case**                          |
|------------------|---------------------------------------|
| `standard`       | General purpose (English text)        |
| `simple`         | Lowercase, split on non-letters       |
| `whitespace`     | Split on whitespace only              |
| `keyword`        | No tokenization (exact match)         |
| `english`        | English stemming, stop words          |
| `autocomplete`   | Edge n-grams for typeahead            |

## Relevance Scoring (TF-IDF & BM25)

### TF-IDF (Classic Algorithm)

**Formula:**
```
score(doc, query) = Σ (TF * IDF)

TF (Term Frequency): How often term appears in document
IDF (Inverse Document Frequency): How rare the term is across all documents
```

**Example:**
```
Query: "brown fox"
Doc1: "The quick brown fox"
Doc2: "The brown brown dog"
Doc3: "The lazy cat"

For "brown" in Doc1:
TF = 1 / 4 = 0.25  (1 occurrence, 4 total words)
IDF = log(3 / 2) = 0.18  (3 total docs, 2 contain "brown")
TF-IDF = 0.25 * 0.18 = 0.045

For "brown" in Doc2:
TF = 2 / 4 = 0.5
IDF = 0.18
TF-IDF = 0.5 * 0.18 = 0.09  (Higher score - term appears more)
```

### BM25 (Elasticsearch Default)

**Improved TF-IDF:**
- Diminishing returns for term frequency (prevents keyword stuffing)
- Document length normalization (long docs don't always win)

```
BM25(doc, query) = Σ IDF(term) * (TF * (k1 + 1)) / (TF + k1 * (1 - b + b * (docLength / avgDocLength)))

k1 = 1.2 (term frequency saturation, typical value)
b = 0.75 (length normalization, typical value)
```

**Why BM25 is better:**
```
Doc1: "fox" (1 occurrence, 10 words)
Doc2: "fox fox fox fox fox" (5 occurrences, 5 words)

TF-IDF: Doc2 scores 5x higher (linear with term count)
BM25: Doc2 scores only ~2x higher (diminishing returns)
```

## Full-Text Search Queries

### Match Query (Most Common)

```json
GET /products/_search
{
  "query": {
    "match": {
      "name": "iphone 15 pro"
    }
  }
}

// Internally becomes: "iphone" OR "15" OR "pro"
// Returns documents containing any of these terms
```

### Match Phrase Query (Exact Order)

```json
GET /products/_search
{
  "query": {
    "match_phrase": {
      "description": "fast charging"
    }
  }
}

// Only returns documents with "fast" immediately followed by "charging"
```

### Multi-Match (Search Multiple Fields)

```json
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "iphone",
      "fields": ["name^3", "description", "category"]  // ^3 = boost name 3x
    }
  }
}
```

### Bool Query (Complex Logic)

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "category": "phones" } }
      ],
      "should": [
        { "match": { "name": "iphone" } },
        { "match": { "name": "samsung" } }
      ],
      "must_not": [
        { "match": { "name": "refurbished" } }
      ],
      "filter": [
        { "range": { "price": { "lte": 1000 } } }
      ]
    }
  }
}

// must: Must match (affects score)
// should: Should match (boosts score if matched)
// must_not: Must not match (excludes documents)
// filter: Must match (doesn't affect score, cached)
```

## Autocomplete / Typeahead

### Edge N-Grams

```
Input: "quick"

Edge n-grams (1-5 chars):
q
qu
qui
quic
quick
```

**Mapping:**
```json
PUT /products
{
  "settings": {
    "analysis": {
      "analyzer": {
        "autocomplete": {
          "tokenizer": "autocomplete_tokenizer",
          "filter": ["lowercase"]
        }
      },
      "tokenizer": {
        "autocomplete_tokenizer": {
          "type": "edge_ngram",
          "min_gram": 1,
          "max_gram": 10,
          "token_chars": ["letter", "digit"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "autocomplete",
        "search_analyzer": "standard"  // Don't n-gram the search query!
      }
    }
  }
}
```

**Query:**
```json
GET /products/_search
{
  "query": {
    "match": {
      "name": "iph"  // Matches "iPhone", "iPhones", etc.
    }
  }
}
```

## Fuzzy Search (Typo Tolerance)

### Levenshtein Distance

```
"iphone" vs "ipohne"
Changes needed: 2 (swap 'h' and 'o')

"iphone" vs "iphone15"
Changes needed: 2 (insert '1' and '5')
```

**Elasticsearch Fuzzy Query:**
```json
GET /products/_search
{
  "query": {
    "fuzzy": {
      "name": {
        "value": "ipohne",
        "fuzziness": "AUTO"  // AUTO = distance 2 for words > 5 chars
      }
    }
  }
}
```

**Match with Fuzziness:**
```json
GET /products/_search
{
  "query": {
    "match": {
      "name": {
        "query": "ipohne 15",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

## Synonyms

```json
PUT /products
{
  "settings": {
    "analysis": {
      "filter": {
        "my_synonym_filter": {
          "type": "synonym",
          "synonyms": [
            "phone, mobile, smartphone",
            "tv, television",
            "laptop, notebook"
          ]
        }
      },
      "analyzer": {
        "synonym_analyzer": {
          "tokenizer": "standard",
          "filter": ["lowercase", "my_synonym_filter"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "synonym_analyzer"
      }
    }
  }
}

// Query: "mobile" will also match "phone" and "smartphone"
```

## Aggregations (Faceted Search)

```json
GET /products/_search
{
  "size": 0,  // Don't return documents, just aggregations
  "aggs": {
    "categories": {
      "terms": {
        "field": "category.keyword",  // Must be keyword type
        "size": 10
      }
    },
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 100 },
          { "from": 100, "to": 500 },
          { "from": 500, "to": 1000 },
          { "from": 1000 }
        ]
      }
    },
    "avg_price": {
      "avg": {
        "field": "price"
      }
    }
  }
}

// Response:
{
  "aggregations": {
    "categories": {
      "buckets": [
        { "key": "phones", "doc_count": 1523 },
        { "key": "laptops", "doc_count": 842 }
      ]
    },
    "price_ranges": {
      "buckets": [
        { "key": "*-100", "doc_count": 234 },
        { "key": "100-500", "doc_count": 1523 }
      ]
    },
    "avg_price": {
      "value": 567.89
    }
  }
}
```

## Performance Optimization

### 1. Use Filters for Exact Matches

```json
// Slow (scores every document)
{
  "query": {
    "match": { "status": "active" }
  }
}

// Fast (cached, no scoring)
{
  "query": {
    "bool": {
      "filter": {
        "term": { "status": "active" }
      }
    }
  }
}
```

### 2. Limit Result Size

```json
GET /products/_search
{
  "size": 10,  // Only return 10 results
  "from": 0,   // Pagination offset
  "_source": ["name", "price"]  // Only return these fields
}
```

### 3. Use Routing

```json
// Index with routing
PUT /products/_doc/123?routing=user_456
{ "name": "iPhone", "user_id": "user_456" }

// Search with routing (searches only 1 shard)
GET /products/_search?routing=user_456
{
  "query": { "match": { "user_id": "user_456" } }
}
```

### 4. Index vs Search Time Boosting

```json
// Index-time boosting (slower indexing, faster search)
PUT /products/_doc/123
{
  "name": "iPhone 15 Pro",
  "name_boost": {
    "value": "iPhone 15 Pro",
    "boost": 2.0
  }
}

// Search-time boosting (preferred)
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "iphone",
      "fields": ["name^2", "description"]
    }
  }
}
```

## Scaling Search

### Horizontal Scaling

```
3 nodes, 3 shards, 1 replica:

Total capacity: 3 nodes * (shard size)
Search QPS: 3x (each node handles queries in parallel)

Adding node 4:
- Elasticsearch rebalances shards automatically
- Search QPS increases
```

### Index-Time vs Query-Time Tradeoffs

| **Strategy**               | **Index Time** | **Query Time** | **Use Case**                |
|----------------------------|----------------|----------------|-----------------------------|
| More shards                | Slower         | Faster         | Large datasets              |
| Fewer replicas             | Faster         | Slower         | Write-heavy workloads       |
| Disable refresh            | Faster         | N/A            | Bulk imports                |
| Pre-compute aggregations   | Slower         | Faster         | Dashboards, analytics       |

### Bulk Indexing

```json
POST /_bulk
{ "index": { "_index": "products", "_id": "1" } }
{ "name": "iPhone 15", "price": 999 }
{ "index": { "_index": "products", "_id": "2" } }
{ "name": "Samsung S23", "price": 899 }
{ "index": { "_index": "products", "_id": "3" } }
{ "name": "Google Pixel 8", "price": 699 }

// 1000x faster than individual indexing
```

## Search Ranking Strategies

### Boosting Recent Documents

```json
GET /articles/_search
{
  "query": {
    "function_score": {
      "query": { "match": { "title": "elasticsearch" } },
      "functions": [
        {
          "gauss": {
            "publish_date": {
              "origin": "now",
              "scale": "30d",
              "decay": 0.5
            }
          }
        }
      ]
    }
  }
}

// Recent articles score higher
```

### Custom Scoring

```json
GET /products/_search
{
  "query": {
    "function_score": {
      "query": { "match": { "name": "iphone" } },
      "functions": [
        {
          "field_value_factor": {
            "field": "popularity",  // Product's popularity score
            "factor": 1.2,
            "modifier": "log1p"
          }
        }
      ],
      "boost_mode": "multiply"
    }
  }
}

// Final score = text_score * log(1 + popularity * 1.2)
```

## Real-World Architecture

### E-commerce Search (Amazon-like)

```
User Query: "wireless headphones under $100"
    ↓
API Gateway
    ↓
Search Service (Application Layer)
    ↓
Elasticsearch Cluster (6 nodes)
    ├── Index: products (20M documents, 10 shards, 2 replicas)
    ├── Filters: price <= 100, category = "headphones"
    ├── Boosting: popularity, ratings, sales
    └── Aggregations: brands, price ranges, ratings
    ↓
Results (50ms p95 latency)
```

### Multi-Index Search

```json
GET /products,reviews,questions/_search
{
  "query": {
    "multi_match": {
      "query": "noise cancelling",
      "fields": ["product_name", "review_text", "question_text"]
    }
  }
}
```

## Elasticsearch Alternatives

| **System**      | **Best For**                        | **Trade-offs**                  |
|-----------------|-------------------------------------|---------------------------------|
| Solr            | Complex queries, faceting           | Harder to scale than ES         |
| Algolia         | Managed SaaS, typo tolerance        | Expensive, vendor lock-in       |
| Typesense       | Lightweight, fast autocomplete      | Smaller community               |
| Meilisearch     | Easy setup, great defaults          | Less mature                     |
| OpenSearch      | Elasticsearch fork (AWS)            | Community smaller than ES       |

## Common Pitfalls

1. **Too many shards**
   - ❌ 100 shards for 10GB index
   - ✅ 2-3 shards for 10GB index

2. **Searching keyword fields with match query**
   - ❌ `"match": { "status.keyword": "active" }`
   - ✅ `"term": { "status.keyword": "active" }`

3. **Deep pagination**
   - ❌ `"from": 10000, "size": 10` (slow!)
   - ✅ Use `search_after` for deep pagination

4. **Not using filters**
   - ❌ Everything in `must` (scored)
   - ✅ Exact matches in `filter` (cached)

5. **Ignoring shard size**
   - ❌ 200GB shards (very slow)
   - ✅ 20-50GB shards (optimal)

## Key Takeaways

1. **Inverted index** is the secret sauce (maps terms → documents)
2. **BM25** balances term frequency and document length
3. **Sharding** enables horizontal scale (search all shards in parallel)
4. **Filters are fast** (no scoring, cacheable)
5. **Analyzers matter** (tokenization determines what's searchable)
6. **Use bulk API** for indexing (1000x faster)
7. **Elasticsearch is eventually consistent** (refresh interval = 1s)
