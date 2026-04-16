# Search Systems — Quick-Fire Interview Q&A

---

**Q1. What is an inverted index and why is it essential for full-text search?**

An inverted index maps each unique term to the list of documents that contain it (a posting list). Given a query term, a lookup is O(1) in the term dictionary — no full table scan required. Without it, finding documents containing a word requires scanning every document. With it, the search is as fast as reading the posting list for each query term and intersecting/scoring the results.

---

**Q2. Walk through the text analysis pipeline for indexing the string "The Quick Brown Fox."**

Tokenize → ["The", "Quick", "Brown", "Fox"]. Lowercase → ["the", "quick", "brown", "fox"]. Remove stop words → ["quick", "brown", "fox"] (remove "the"). Apply stemmer → ["quick", "brown", "fox"]. The same pipeline must run on the query "QUICK BROWN" → ["quick", "brown"] — analysis symmetry ensures query terms match indexed terms.

---

**Q3. What is the difference between stemming and lemmatization?**

Stemming applies heuristic rules to chop word endings (Porter: "running" → "run", "studies" → "studi" — may produce non-words). Lemmatization uses a dictionary to find the canonical base form ("better" → "good", "running" → "run" — always real words). Stemming is faster; lemmatization is linguistically correct. Elasticsearch's default `english` analyzer uses Porter stemmer.

---

**Q4. Why is TF-IDF's term frequency score considered a problem, and how does BM25 address it?**

TF-IDF's TF score grows linearly and without bound — a document mentioning "cat" 100 times scores 100× higher than a document mentioning it once, even if both are clearly about cats. BM25 applies TF saturation: the contribution of additional occurrences diminishes and asymptotes at k₁+1 (≈2.2 with k₁=1.2). Beyond ~10 occurrences, adding more has negligible effect. This better matches human relevance judgments.

---

**Q5. What does the `b` parameter in BM25 control?**

`b` (0–1) controls length normalization. At `b=0`, document length is ignored. At `b=1`, TF is fully normalized by document length (a 1-page doc and a 100-page doc are equally weighted per term). The default `b=0.75` provides partial normalization — longer documents are somewhat penalized for having more term occurrences by chance. This prevents encyclopedias from dominating search results over focused articles.

---

**Q6. How does a query fan out across Elasticsearch shards?**

In two phases: (1) Scatter — the coordinating node sends the query to all shards; each shard runs Lucene locally and returns top-K document IDs + scores. (2) Gather — the coordinating node merges all local top-K lists into a global top-K, then fetches the full source documents from their respective shards. Only `size × num_shards` doc IDs flow in phase 1; full documents transfer only in phase 2.

---

**Q7. What is "near real-time" search in Elasticsearch and what causes the delay?**

NRT means documents are searchable within about 1 second of being indexed. The delay is the `refresh_interval` (default 1s). Indexed documents land in an in-memory buffer first. On each refresh, the buffer is flushed to a new Lucene segment on disk — only after this flush are the documents searchable. Forcing `refresh=true` per request makes documents immediately searchable but is expensive and should not be done at high write rates.

---

**Q8. Why are there two types of replica strategies in Elasticsearch — query-time replicas vs. shard replicas?**

Shard replicas (configured at index level) add redundant copies of each primary shard. Reads can be served from any replica, distributing read load. This is the standard replica strategy. There are no "query-time replicas" in Elasticsearch by default — this is a distractor. The main mechanisms are: (1) replica shards for HA + read scaling, (2) coordinating nodes that route but don't store data, and (3) cross-cluster search for querying multiple clusters.

---

**Q9. What is the difference between `filter` and `must` in a `bool` query?**

Both `filter` and `must` clauses must match for a document to be returned. The difference: `must` clauses contribute to the relevance score (BM25 scoring runs). `filter` clauses are binary — they only include/exclude, and their results are cached by Elasticsearch. Use `filter` for structured conditions (status, date range, category) and `must` for full-text search terms. Mixing both gives relevance-ranked results that also meet hard constraints.

---

**Q10. What is Lucene's segment architecture and why are segments immutable?**

A Lucene index is made of multiple segment files. Each segment is an independent inverted index. Segments are immutable — writes create new segments; updates mark the old version deleted in the old segment and write the new version to a new segment. Immutability enables: simple cache validity (segment content never changes), safe concurrent reads, and efficient OS-level file caching. Periodic segment merges compact small segments and clean up deleted docs.

---

**Q11. Why does deep pagination (`from: 10000, size: 10`) harm Elasticsearch performance?**

Every shard must materialize and sort the top `from + size` results, send them to the coordinator, which then picks the top `size` from the global list. At `from=10000, size=10`, each shard sends 10,010 items to the coordinator — even though only 10 are returned. With 5 shards, coordinator receives 50,050 items, sorts, discards 50,040. Use `search_after` (cursor-based pagination using the last hit's sort values) for deep pagination — each shard only sends `size` items regardless of page depth.

---

**Q12. What is the Completion Suggester in Elasticsearch and how does it differ from a wildcard query?**

The Completion Suggester uses an in-memory FST (Finite State Transducer) data structure pre-built for prefix lookups. It's extremely fast (sub-millisecond for type-ahead). A `wildcard` query (`*iphone*`) runs against the inverted index — it can match any position in the term but requires scanning the term dictionary, which is much slower. For autocomplete/type-ahead, always use the Completion Suggester; use wildcard only for suffix/infix matching where Completion is insufficient.

---

**Q13. What is the `_source` field and when would you disable it?**

`_source` stores the original JSON document verbatim. By default it's enabled and returned with every search hit. Disabling `_source: false` saves disk space (~40–60% reduction for text-heavy indexes) but means you cannot retrieve the original document from ES — a re-index from the original data source is required to get it back. A middle ground: use `includes` in the mapping to store only specific fields, or use `store: true` on individual fields.

---

**Q14. Explain the trade-off between number of shards and search performance.**

More shards = more parallelism on write (each shard handles a fraction of indexing) and more fan-out on query (each shard is queried independently). But more shards means: higher overhead per query (JVM thread per shard), more small segment files, more heap usage for shard metadata. Rule of thumb: size shards at 20–40 GB; don't exceed dozens of shards per index unless you have tens of nodes. Over-sharding is a common Elasticsearch performance antipattern.

---

**Q15. What is hybrid search and why is it becoming the standard for modern search?**

Hybrid search combines BM25 (keyword-based relevance) with vector similarity search (semantic/neural relevance). BM25 excels at exact keyword matches ("iPhone 15 Pro") but fails for semantic queries ("best phone for photography"). Vector search (using sentence embeddings) captures semantic meaning but misses exact matches. Combining both via Reciprocal Rank Fusion (RRF) or a weighted linear combination gives better results than either alone. Elasticsearch 8.x supports HNSW-based KNN vector search natively, enabling hybrid search in a single query.
