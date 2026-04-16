# Search Systems — Notes & Reference Sheet

## 1. Inverted Index Anatomy

```
Term dictionary (FST):      Posting lists:
"apple"  ──────────────►  [(doc_3, tf=2, pos=[1,7]), (doc_9, tf=1, pos=[4])]
"banana" ──────────────►  [(doc_1, tf=1, pos=[2]), (doc_3, tf=3, pos=[2,5,8])]
"cherry" ──────────────►  [(doc_7, tf=2, pos=[0,3])]
```

**Posting list fields:**
- `doc_id`: which document
- `term_freq (tf)`: how many times the term appears
- `positions`: byte offsets or token positions (needed for phrase search)

**Building:** tokenize → lowercase → stop words → stem → insert into posting lists

---

## 2. Text Analysis Pipeline Reference

```
Input text
    │
    ▼  Tokenizer (splits on whitespace, punctuation)
["Hello", "world", "it's", "great"]
    │
    ▼  Character filter (optional: strip HTML, normalize unicode)
    │
    ▼  Token filters (applied in order):
    │   ├── lowercase:      ["hello", "world", "it's", "great"]
    │   ├── stop words:     ["hello", "world", "great"]   (remove "it's")
    │   ├── synonyms:       ["hello", "world", "excellent"] (great→excellent)
    │   └── stemmer:        ["hello", "world", "excel"]
    ▼
Indexed tokens: { "hello", "world", "excel" }
```

**Elasticsearch built-in analyzers:**
- `standard`: whitespace tokenize + lowercase (no stemming)
- `english`: standard + stop words + Porter stem + possessive removal
- `keyword`: no analysis — stores exact value (for `keyword` fields)

---

## 3. TF-IDF vs. BM25 Formula Comparison

**TF-IDF:**
$$\text{score}(t, d) = \frac{\text{count}(t, d)}{|d|} \times \log\left(\frac{N}{1 + \text{df}(t)}\right)$$

**BM25 (k₁=1.2, b=0.75):**
$$\text{score}(t, d) = \text{IDF}(t) \times \frac{\text{tf} \cdot (k_1+1)}{\text{tf} + k_1\left(1 - b + b\cdot\frac{|d|}{\text{avgdl}}\right)}$$

| Aspect | TF-IDF | BM25 |
|---|---|---|
| TF growth | Linear, unbounded | Saturates at k₁+1 |
| Document length | No normalization | Normalized by avgdl, controlled by b |
| Performance | Good baseline | Better for natural language |
| Default in ES | Before 5.x | 5.x+ (current default) |

**Key parameters:**
- `k₁ = 1.2`: TF saturation — higher = more weight to term frequency
- `b = 0.75`: length normalization — `b=1` = full normalization; `b=0` = no length norm

---

## 4. Elasticsearch Architecture Quick Reference

```
Cluster
  ├── Master node (1 elected): cluster state, shard allocation
  ├── Data nodes (N): store shards, execute queries
  └── Coordinating nodes (optional): route requests only, no data

Index "products" (3 primary shards, 1 replica each = 6 shards total)
  ├── Node-1: Primary-0, Replica-1
  ├── Node-2: Primary-1, Replica-2
  └── Node-3: Primary-2, Replica-0

Shard assignment:
  shard = murmurhash(doc_id) % num_primary_shards
```

**Near Real-Time (NRT):**
- Writes go to in-memory buffer
- `refresh_interval: 1s` → buffer flushed to Lucene segment → searchable
- `fsync` to disk happens on `flush` (every ~30 min or when buffer is full)

**Segment lifecycle:**
```
Write → in-memory buffer → segment (searchable) → periodic merge → large segment
```
Merging reclaims space from deleted/updated docs (they're marked deleted, not removed).

---

## 5. Query DSL Cheat Sheet

```json
// Match (full-text, analyzed)
{"match": {"title": "elasticsearch"}}

// Multi-field with boost
{"multi_match": {"query": "elasticsearch", "fields": ["title^3", "description"]}}

// Exact match (not analyzed)
{"term": {"status": "published"}}

// Range
{"range": {"price": {"gte": 100, "lte": 500}}}

// Boolean compound
{"bool": {
  "must":     [...],  // Scored (must match)
  "filter":   [...],  // Not scored, cached (must match)
  "should":   [...],  // Scored (optional, boosts score)
  "must_not": [...]   // Not scored (must not match)
}}

// Phrase (exact word order)
{"match_phrase": {"title": "quick brown fox"}}

// Phrase with slop (gap allowed)
{"match_phrase": {"title": {"query": "quick fox", "slop": 1}}}
```

**Filter vs. must:**
- `filter`: binary inclusion only, no scoring. Results cached. ~10–100× faster for static values.
- `must`: contributes to relevance score. Use for text search terms only.

---

## 6. Aggregation Types Reference

| Aggregation | Type | Use |
|---|---|---|
| `terms` | Bucket | Top N values of a field (facets) |
| `range` | Bucket | Count docs in price/age ranges |
| `date_histogram` | Bucket | Count events per day/week/month |
| `filter` | Bucket | Apply a query, then aggregate within it |
| `avg`, `sum`, `min`, `max` | Metric | Average price, total revenue |
| `stats` | Metric | min+max+avg+sum+count at once |
| `cardinality` | Metric | Approximate distinct count (HyperLogLog) |
| `percentiles` | Metric | P50, P95, P99 latency |

---

## 7. Relevance Tuning Reference

| Technique | ES Config | When |
|---|---|---|
| Field boost | `"title^3"` | Title match > body match |
| Function score | `function_score` with `field_value_factor` | Boost by review count |
| Synonyms | `synonym` token filter | Domain vocabulary |
| Completion suggester | `type: completion` field | Autocomplete prefix |
| Decay functions | `gauss`, `exp`, `linear` on a date/geo field | Boost recent or nearby |

---

## 8. Common Elasticsearch Settings

| Setting | Default | Notes |
|---|---|---|
| `number_of_shards` | 1 (7.x+) | Cannot change after index creation |
| `number_of_replicas` | 1 | Can change at runtime |
| `refresh_interval` | 1s | Increase to 30s during bulk indexing |
| `index.max_result_window` | 10,000 | Cost of deep pagination: use `search_after` instead |
| `index.mapping.total_fields.limit` | 1,000 | Prevent mapping explosion |

**Bulk indexing optimization:**
```python
es.index(refresh="false")    # Don't refresh per-document
es.indices.put_settings({"refresh_interval": "30s"})
# ... bulk index with helpers.bulk() ...
es.indices.put_settings({"refresh_interval": "1s"})
es.indices.refresh()
```

---

## 9. Key Numbers

| Metric | Rule of thumb |
|---|---|
| Shard size | 10–50 GB (sweet spot: 20–40 GB) |
| Primary shards | Set at creation — can't change |
| NRT latency | Up to 1 second (refresh_interval) |
| `max_result_window` | 10,000 — use `search_after` beyond |
| field cardinality for `terms` agg | OK up to millions (uses heap) |
| Autocomplete prefix length | Start from 2–3 chars for good UX |

---

## 10. When NOT to Use Elasticsearch

| Need | Better choice |
|---|---|
| ACID transactions | PostgreSQL |
| Primary data store | PostgreSQL/DynamoDB |
| Simple prefix/exact search | DB index with LIKE 'x%' |
| Real-time analytics on structured data | ClickHouse, BigQuery |
| ML ranking at query time | Learning-to-rank plugin + feature store |

**Elasticsearch is best for:** full-text search, log analytics (ELK stack), faceted search, autocomplete, and geospatial search.
