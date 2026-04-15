# Search Systems — Tricky Interview Questions

---

## Question 1: "You say Elasticsearch is 'near real-time' — 1-second delay. Our product manager says we need zero-delay search — documents should be searchable the instant they're written. How do you achieve this?"

**The trap:** Assuming refresh must be called per-document or that the 1s limit is absolute.

**Strong answer:**

There are two approaches, and the right choice depends on the use case:

**Option 1: Force refresh per write (bad for throughput):**
```python
es.index(index="my-index", document=doc, refresh=True)
# Document immediately searchable — but synchronous segment flush costs ~5–50ms per write
```
At 1,000 writes/second this would require 1,000 segment flushes/second — prohibitive. Each flush creates a new Lucene segment; too many small segments degrade query performance.

**Option 2: Reduce refresh_interval (better, with limits):**
```json
PUT /my-index/_settings
{ "refresh_interval": "100ms" }
```
Reduces latency to 100ms at the cost of more frequent flushes and IO. Still not zero.

**Option 3: Challenge the requirement:**
For most real products, 1-second latency is imperceptible to users. "Zero-delay" requirements often come from conflating write confirmation with search visibility. A more useful question: "What is the actual user-visible consequence of a 1s delay?" If negligible, keep the 1s default.

**Option 4: Dual-write with a different store for instant lookups:**
For the specific case of "user can immediately search their own just-written document" (read-your-writes): write to both Elasticsearch AND a Redis key-value store. The user's own documents are immediately available via a Redis lookup; the search index catches up within 1 second for everyone else.

**Key insight:** If you truly need instant indexing at scale, you're pushing against Elasticsearch's design. Consider Apache Kafka + Materialize, or a database with full-text search (PostgreSQL `tsvector`) designed for mutable, immediately-consistent writes.

---

## Question 2: "You query Elasticsearch for 'elasticsearch' and want the top 10 results. But you have 5 shards. Each shard returns its local top 10. Doesn't that mean you might miss a globally relevant document that was ranked 11th on one shard?"

**The trap:** The scatter-gather protocol with fixed `size` per shard accumulates results that could miss globally relevant docs from shards that returned a different local top-10.

**Strong answer:**

Yes — this is a real problem called the "ranking inconsistency in distributed search."

Each shard computes its local BM25 scores independently. A document ranked 11th on shard-3 might have a higher global score than something ranked in the top 10 on shard-1, but it never gets sent to the coordinator.

**Mitigation: `dfs_query_then_fetch` search type:**

The default `query_then_fetch` uses each shard's local IDF. The same term's IDF differs shard-to-shard if the document distribution is uneven.

`dfs_query_then_fetch` adds a pre-flight step: the coordinator first collects term statistics (df, N) from all shards, computes global IDF values, then broadcasts them to all shards for the query phase. All shards use the same IDF → globally consistent BM25 scores → no ranking inconsistency.

**Trade-off:**
- `query_then_fetch` (default): N round-trips per shard (1), faster, statistics may be inconsistent across shards
- `dfs_query_then_fetch`: 2N round-trips per shard, slower (~30–50% overhead), globally consistent

**Practical guidance:**
- Large indexes (millions of docs per shard): document distribution across shards is roughly uniform → per-shard IDF ≈ global IDF → `query_then_fetch` is fine
- Small indexes or very uneven routing: use `dfs_query_then_fetch`
- Development/testing: always use `dfs_query_then_fetch` to avoid confusing ranking bugs

---

## Question 3: "A product manager says: 'Our search results aren't good enough — users complain that popular products rank below obscure ones.' BM25 is correctly configured. What's wrong?"

**The trap:** BM25 only measures textual relevance — it knows nothing about business signals like sales, clicks, or ratings.

**Strong answer:**

BM25 measures textual relevance only: "does this document match the query?" It has no concept of product quality, popularity, or user preferences. A product with a 2-star rating and a well-keyword-optimized description can outrank a 5-star bestseller with a brief description.

**Solutions — incorporating business signals:**

**1. Function Score Query (multiply BM25 by a business metric):**
```json
{
  "query": {
    "function_score": {
      "query": {"multi_match": {"query": "running shoes", "fields": ["title", "desc"]}},
      "functions": [
        {
          "field_value_factor": {
            "field": "review_count",
            "factor": 0.1,
            "modifier": "log1p",   // log(1 + review_count) prevents huge values
            "missing": 1
          }
        },
        {
          "field_value_factor": {
            "field": "rating",
            "factor": 2.0,
            "modifier": "none"
          }
        }
      ],
      "boost_mode": "multiply",
      "score_mode": "multiply"
    }
  }
}
```

**2. Decay functions — boost recent products:**
```json
{
  "gauss": {
    "created_at": { "origin": "now", "scale": "30d", "decay": 0.5 }
  }
}
```
Products created within 30 days get a boost that decays exponentially with age.

**3. Learning to Rank (L2R):**
For sophisticated relevance tuning: collect user interaction data (click-through rates, conversions), train a gradient-boosted model with features (BM25 score, rating, review count, price, brand quality score), and use Elasticsearch's `ltr` plugin to apply the model at query time.

**4. Pinned queries:**
Merchandising override — pin specific product IDs to the top for high-value queries regardless of BM25:
```json
{"pinned": {"ids": ["prod_123", "prod_456"], "organic": {"match": {"title": "running shoes"}}}}
```

---

## Question 4: "You're showing 'Did you mean X?' suggestions when a user types a query with no results. How would you implement this in Elasticsearch?"

**The trap:** Candidates say "use the Completion Suggester" — wrong tool. Completion Suggester is for autocomplete on existing indexed values, not for misspelling correction.

**Strong answer:**

The Completion Suggester matches prefixes from an FST of known terms — it doesn't handle misspellings. For "Did you mean?" (fuzzy spell correction), use the **Term Suggester** or **Phrase Suggester**:

**Term Suggester — per-word correction:**
```json
{
  "suggest": {
    "spell_check": {
      "text": "elasticsearh distributed",
      "term": {
        "field": "title",
        "suggest_mode": "missing",   // only suggest if no results for that term
        "min_word_length": 4,
        "max_edits": 2               // max edit distance (Levenshtein)
      }
    }
  }
}
```
Returns: `"elasticsearh"` → suggest `"elasticsearch"` (1 edit away)

**Phrase Suggester — whole-phrase correction:**
```json
{
  "suggest": {
    "my_suggester": {
      "text": "elasticsearh distribued",
      "phrase": {
        "field": "title.trigram",   // requires trigram tokenizer on field
        "max_errors": 2,
        "direct_generator": [{"field": "title.trigram", "suggest_mode": "always"}],
        "highlight": {"pre_tag": "<em>", "post_tag": "</em>"}
      }
    }
  }
}
```
Returns: "Did you mean: <em>elasticsearch</em> <em>distributed</em>?"

**Implementation flow:**
1. Run query — if `hits.total.value == 0`
2. Run Term/Phrase Suggester on the same query text
3. If suggestion found: display "Did you mean X?" with a link to re-run with the corrected query
4. Optionally auto-run the corrected query in the background and show it ("Showing results for X instead")

---

## Question 5: "You have 100 million product documents. An Elasticsearch query takes 200ms. How do you optimize it?"

**The trap:** Vague optimization question — tests structured diagnostic thinking.

**Strong answer:**

I'd diagnose before optimizing. Use `_search/profile: true` to see where time is spent (query parsing, scoring, fetch). Then apply targeted fixes:

**1. Filters for non-scored conditions (biggest win):**
```json
// BAD: status in must clause → scored but status doesn't affect relevance
{"bool": {"must": [{"term": {"status": "active"}}, {"match": {"title": "..."}}]}}

// GOOD: status in filter → cached, binary, no scoring overhead
{"bool": {"must": [{"match": {"title": "..."}}], "filter": [{"term": {"status": "active"}}]}}
```
Filter results are cached — subsequent identical filters are nearly free.

**2. Reduce scope with routing:**
If products are by vendor, route queries for vendor X directly to the shard(s) containing vendor X data: `GET /products/_search?routing=vendor_123`. Queries hit 1 shard instead of all → (N-1)/N less work.

**3. Limit `_source` fields returned:**
```json
{"_source": ["title", "price", "image_url"]}  // Don't return 20-field source when you need 3
```

**4. Use `search_after` instead of `from/size` for pagination beyond page 2.**

**5. Size shards correctly:**
If shards are too small (< 5 GB), there are too many of them → too much fan-out overhead per query. Merge indexes with `_rollup` or `reindex`. Target 20–40 GB per shard.

**6. Index mapping optimization:**
- Disable `_source` on fields never retrieved
- Set `index: false` on fields never queried
- Set `doc_values: false` on text fields not used for aggregations/sorting

**7. Hardware:**
- SSD: Lucene I/O heavy — SSD reduces segment read latency significantly
- Heap: ES recommends half of available RAM for heap (max 30 GB — above that, JVM compressed OOPs break)

---

## Question 6: "Why doesn't Elasticsearch guarantee search results are complete? Could a document ever be totally lost from search results?"

**The trap:** Tests deep understanding of Lucene's write path and Elasticsearch's durability model.

**Strong answer:**

Yes, there are scenarios where a document could be invisible or lost, and they're worth knowing:

**1. In-flight documents (known and expected):** Documents in the in-memory buffer (not yet refreshed) are not searchable. This is the NRT "feature," not a bug — lasts at most `refresh_interval` (1s by default).

**2. Unacknowledged replication (potential data loss):** Elasticsearch writes to the primary shard first, then replicates to replicas. With `wait_for_active_shards=1` (only primary must ack), a crash of the primary before replication completes means the document is lost — the replica never got it. Solution: set `wait_for_active_shards=all` for critical writes, or at minimum `quorum`.

**3. Translog and fsync timing:** The translog (Write-Ahead Log) durably records each write before the in-memory buffer. By default, translog is fsynced every 5 seconds (`index.translog.durability: async`). If the node crashes within that 5-second window, the translog may be incomplete → data loss. Setting `index.translog.durability: request` fsyncs on every write but has significant performance cost.

**4. Split brain/unrecoverable corruption (rare):** If cluster state is corrupted during network partition recovery, old data may be overwritten by stale replicas being promoted. Fixed in modern Elasticsearch by cluster coordination via Raft-like protocol (since 7.x — replaced Zen discovery).

**Practical answer for interviews:** For most applications, NRT is the expected behavior. For financial/critical data, set `wait_for_active_shards` appropriately and consider `translog.durability: request`. For truly critical writes, Elasticsearch is not your source of truth — a database is.
