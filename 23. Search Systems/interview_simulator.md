# Search Systems — Interview Simulator

---

## Scenario 1: Design a Product Search for an E-Commerce Platform (Amazon-Like)

**Interviewer prompt:**

> "Design the search system for an e-commerce platform with 500 million products, 100 million daily searches, and requirements for: full-text relevance, faceted filtering (category/brand/price), real-time inventory status, and autocomplete. Walk through your architecture."

---

### Strong Answer

**Scale estimates:**
- 500M products × ~2KB per indexed document = ~1 TB of index data
- 100M searches/day ≈ 1,200 QPS average, ~5,000 QPS peak
- Write throughput: millions of inventory updates/day (in_stock, price changes)

**Index design — what to store in Elasticsearch:**

Denormalized document per product:
```json
{
  "product_id": "p_123",
  "title": "Nike Air Max 270 Running Shoes",
  "description": "Lightweight running shoe with Max Air cushioning...",
  "brand": "Nike",
  "category": "shoes/running",
  "price": 129.99,
  "rating": 4.6,
  "review_count": 3421,
  "in_stock": true,
  "tags": ["running", "lightweight", "air-cushion"],
  "title_suggest": "Nike Air Max 270 Running Shoes"
}
```

**Field mapping decisions:**
- `title`: `text` (analyzed, Porter stem) + `keyword` (exact match, sorting) + `completion` (autocomplete)
- `category`: `keyword` (exact, aggregations, filtering — no analysis)
- `price`, `rating`: `float` (range queries)
- `in_stock`: `boolean` (filter — never contributes to score)
- `description`: `text` only (no keyword — too large)

**Sharding:**
- 1 TB data → target 20–40 GB/shard → ~30 primary shards
- Route by `category` hash (`?routing=shoes`) → category-scoped queries hit 1 shard instead of all 30

**Query design — relevance + filters:**
```json
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": [
            {"multi_match": {
              "query": "nike running shoes",
              "fields": ["title^4", "brand^3", "description^1"],
              "operator": "or"
            }}
          ],
          "filter": [                                    // Scored=false, cached
            {"term": {"in_stock": true}},
            {"term": {"category": "shoes/running"}},
            {"range": {"price": {"lte": 200}}}
          ]
        }
      },
      "functions": [
        {
          "field_value_factor": {                        // Boost popular products
            "field": "review_count",
            "modifier": "log1p",
            "factor": 0.5
          }
        },
        {
          "field_value_factor": {
            "field": "rating",
            "modifier": "none",
            "factor": 1.5
          }
        }
      ],
      "boost_mode": "multiply"
    }
  },
  "aggs": {
    "brands": {"terms": {"field": "brand", "size": 20}},
    "price_ranges": {"range": {"field": "price",
      "ranges": [{"to": 50}, {"from": 50, "to": 100}, {"from": 100, "to": 200}, {"from": 200}]
    }},
    "avg_rating": {"avg": {"field": "rating"}}
  },
  "size": 20
}
```

**Autocomplete:**
- Use Completion Suggester on `title_suggest` field
- In-memory FST — sub-millisecond completions
- Pre-populate with top-selling/searched products for each prefix

**Inventory updates (in_stock, price):**
- Partial update via `_update` API — only changed fields re-indexed, not full document
- For price/inventory: ~1M updates/day ≈ 12 updates/sec — manageable
- Consider `refresh_interval: 5s` to batch small updates instead of 1s default (trade-off: 5s lag on inventory visibility)

**Architecture:**
```
User query → API Gateway → Search Service → Elasticsearch cluster (30 shards, RF=2)
                                             ↑
PostgreSQL (source of truth)  ────────────► Kafka "product-updates" topic
                                             ↑
Product Service (writes to postgres)         │
                                         Sync Service (consumer: indexes to ES)
```

**Deep pagination:** Use `search_after` (cursor-based) rather than `from/size` beyond page 3.

**Caching:** Cache identical query+filter combinations in Redis (TTL=30s) — many users search for the same popular queries.

---

## Scenario 2: Design a Log Search System (ELK Stack)

**Interviewer prompt:**

> "Your microservices generate 10 TB of logs per day. Engineers need to search logs by service name, severity level, time range, and keyword (e.g., 'NullPointerException'). Mean search latency should be < 2s. Design the system."

---

### Strong Answer

**Key differences from product search:**
- Data is time-series (logs ordered by timestamp)
- Very high write throughput (10 TB/day ≈ 120 MB/sec ingestion)
- Data is mostly immutable — logs don't change after creation
- Time-based access: recent logs queried 99% of the time, old logs rarely accessed
- Scale: 10 TB/day uncompressed → ~2–3 TB stored (Elasticsearch compresses aggressively)

**Ingestion pipeline (Classic ELK):**

```
Service logs → Filebeat (lightweight agent) → Logstash (parse/enrich) → Elasticsearch
                                           (or Fluent Bit for lower overhead)
Alternatively: → Kafka (buffer) → Logstash consumers → Elasticsearch
```

**Why Kafka in the middle?** If Elasticsearch is temporarily slow/down, logs buffer in Kafka (retention=3 days) — no data loss. Back-pressure is absorbed.

**Index strategy — time-based rolling indices:**

```
logs-2024.01.01
logs-2024.01.02
...
logs-2024.04.15  ← current
```

Benefits:
- Query with `index: logs-2024.04.*` for this month → limits fan-out to this month's shards
- Drop old index: `DELETE logs-2024.01.*` — much faster than deleting by query
- ILM (Index Lifecycle Management) automates: Hot → Warm → Cold → Delete

**Index Lifecycle Management (ILM):**
```
Hot  (0–3 days):   3 primary shards, 1 replica, SSD
Warm (3–30 days):  1 primary shard,  1 replica, HDD (merged into larger segments)
Cold (30–90 days): 0 replicas, frozen index, HDD
Delete (>90 days): index removed
```

**Mapping for log documents:**
```json
{
  "timestamp": {"type": "date"},
  "service": {"type": "keyword"},        // exact match, aggregations
  "severity": {"type": "keyword"},       // INFO, WARN, ERROR
  "host": {"type": "keyword"},
  "trace_id": {"type": "keyword"},       // exact correlation ID lookup
  "message": {"type": "text", "analyzer": "standard"},  // full-text search
  "exception_class": {"type": "keyword"}
}
```

**Search query pattern:**
```json
{
  "query": {
    "bool": {
      "filter": [                                   // Not scored — all structural
        {"term": {"service": "payment-service"}},
        {"term": {"severity": "ERROR"}},
        {"range": {"timestamp": {"gte": "now-1h"}}}
      ],
      "must": [                                     // Scored — text search
        {"match": {"message": "NullPointerException"}}
      ]
    }
  },
  "sort": [{"timestamp": {"order": "desc"}}],
  "size": 100
}
```

**Performance optimization for high-write throughput:**
- Bulk indexing: `_bulk` API — 500–5000 documents per request
- `refresh_interval: 30s` during peak ingestion (can tolerate 30s search lag on logs)
- Disable `_source` for fields not needed in results (save disk IO)
- Pre-compute `exception_class` as a keyword field (parsed from message by Logstash) → fast aggregation queries

**Common dashboard query — error rate by service:**
```json
{"aggs": {
  "by_service": {
    "terms": {"field": "service", "size": 20},
    "aggs": {
      "error_count": {"filter": {"term": {"severity": "ERROR"}}},
      "per_minute": {"date_histogram": {"field": "timestamp", "fixed_interval": "1m"}}
    }
  }
}}
```

---

## Scenario 3: Design a Full-Text Document Search Service (Confluence/Notion-Like)

**Interviewer prompt:**

> "Design a search service for a wiki platform. Users can search by keywords, filter by workspace/team, and results must show relevant excerpts (highlighted snippets). New documents should appear in search within 5 seconds. How do you design this?"

---

### Strong Answer

**Requirements breakdown:**
- Full-text search across title + body
- Filter by workspace (multi-tenant isolation)
- Highlighted excerpts showing where match occurs
- 5-second freshness SLA (NRT with reduced refresh interval)
- Access control: users only see documents they're permitted to

**Index design:**

```json
{
  "page_id": "page_123",
  "workspace_id": "ws_456",
  "title": "Elasticsearch Indexing Best Practices",
  "body": "The inverted index is the core...",
  "author_id": "user_789",
  "created_at": "2024-01-15T10:00:00Z",
  "updated_at": "2024-04-01T14:30:00Z",
  "status": "published",
  "tags": ["elasticsearch", "search", "indexing"],
  "allowed_team_ids": ["team_a", "team_b"]   // pre-computed access list
}
```

**Access control approach:** Embed `allowed_team_ids` in the document and filter at query time: `{"terms": {"allowed_team_ids": user.team_ids}}`. Avoids a separate authorization service roundtrip per result, but requires re-indexing when permissions change. For large teams (millions of members), this doesn't scale — use a `workspace_id` filter instead and enforce project-level ACLs in the application layer.

**5-second freshness:**
```json
PUT /wiki-pages/_settings
{ "refresh_interval": "2s" }
```
Set `refresh_interval: 2s` — documents become searchable within 2s (worst case), well within the 5s SLA. At moderate write rates (thousands of new pages/hour), this is sustainable.

**Search with highlighted excerpts:**

```json
{
  "query": {
    "bool": {
      "must": [
        {"multi_match": {
          "query": "inverted index elasticsearch",
          "fields": ["title^4", "body^1"],
          "type": "best_fields"
        }}
      ],
      "filter": [
        {"term": {"workspace_id": "ws_456"}},
        {"term": {"status": "published"}},
        {"terms": {"allowed_team_ids": ["team_a"]}}
      ]
    }
  },
  "highlight": {
    "fields": {
      "title":  {"number_of_fragments": 0},        // Return full title with highlights
      "body":   {"number_of_fragments": 3,          // Return top 3 matching excerpts
                 "fragment_size": 200}
    },
    "pre_tags":  ["<mark>"],
    "post_tags": ["</mark>"]
  },
  "_source": ["page_id", "title", "author_id", "updated_at"],  // Only metadata in _source
  "size": 10
}
```

Highlighting response:
```json
{
  "highlight": {
    "body": [
      "The <mark>inverted index</mark> is the core data structure...",
      "When <mark>Elasticsearch</mark> receives a query..."
    ]
  }
}
```

**Synchronization (wiki DB → Elasticsearch):**

Option 1: Application-level dual-write (simple, tight coupling)
```python
async def update_page(page_id, content):
    await db.update(page_id, content)
    await es.index(index="wiki-pages", id=page_id, document=build_doc(content))
```

Option 2: CDC via Debezium (decoupled, eventual)
- Debezium tails PostgreSQL WAL → Kafka `page-changes` topic
- ES indexer consumes the topic, builds ES document, indexes it
- Delay: ~1–2s end-to-end (well within 5s SLA)
- Resilient: ES downtime doesn't affect writes; Kafka buffers changes

**Delete handling:**
On page deletion, send `delete` to Elasticsearch: `es.delete(index="wiki-pages", id=page_id)`. With Lucene's soft-delete model, the document is marked deleted and removed on next segment merge.

**What interviewers look for:**
- Access control embedded in the document (filter at query time vs. post-filter)
- Highlighting configuration — `number_of_fragments: 0` for title (full field), `>0` for body
- 5-second freshness via `refresh_interval` tuning, not `refresh=true` per write
- Dual-write vs. CDC trade-offs — CDC is correct and decoupled
